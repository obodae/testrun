"""
Bitcoin to Cash App Transfer Tool
==================================

IMPORTANT LEGAL AND SECURITY NOTES:
====================================

1. KYC REQUIREMENTS:
   - Coinbase requires Know Your Customer (KYC) verification
   - You must provide government ID and personal information
   - Unverified accounts have transaction limits
   - Cash App also requires identity verification
   - Both services must be linked to your legal identity

2. TRANSACTION FEES:
   - Coinbase withdrawal fee: typically 0.0005 BTC or varies by payment method
   - Cash App transfer fee: typically $0 for bank transfers, but varies by region
   - Bitcoin network fee: included in Coinbase withdrawal calculation
   - Total cost can be 1-3% of transaction amount

3. SECURITY CONSIDERATIONS:
   - Store API keys in environment variables, NEVER in code
   - Use OAuth 2.0 for production applications
   - Enable 2FA on both Coinbase and Cash App accounts
   - Use IP whitelisting on Coinbase API keys
   - Never share API keys or passphrases
   - Use HTTPS for all connections

4. LEGAL CONSIDERATIONS:
   - Crypto transactions are taxable events
   - Report all conversions to your tax authority
   - Comply with local money transmission regulations
   - Both Coinbase and Cash App file reports (Form 8949, etc.)

5. ACCOUNT SETUP REQUIREMENTS:
   - Valid government-issued ID
   - Proof of address (utility bill, bank statement, etc.)
   - Verified phone number
   - Verified email address
   - Cash App account linked to valid bank account

6. LIMITATIONS:
   - Daily/monthly transaction limits apply
   - Newly verified accounts may have lower limits
   - Holds may be placed on accounts for compliance reviews
   - This tool is for PERSONAL use only, not for businesses (see Coinbase ToS)
"""

import os
import json
import time
import requests
from typing import Dict, List, Optional, Tuple
from datetime import datetime
from decimal import Decimal
import hmac
import hashlib
from urllib.parse import urljoin
import logging

# Set up logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


class BitcoinToCashAppTransfer:
    """
    Securely transfer Bitcoin from Coinbase to Cash App.
    
    This class handles:
    - Bitcoin price conversion to USD
    - Coinbase API authentication
    - Bitcoin withdrawal from Coinbase
    - Bank transfer to linked Cash App account
    - Transaction tracking and monitoring
    - Fee calculations and reporting
    """
    
    def __init__(
        self,
        coinbase_api_key: Optional[str] = None,
        coinbase_api_secret: Optional[str] = None,
        coinbase_passphrase: Optional[str] = None,
        sandbox: bool = False
    ):
        """
        Initialize the transfer handler.
        
        Args:
            coinbase_api_key: Coinbase API key (defaults to COINBASE_API_KEY env var)
            coinbase_api_secret: Coinbase API secret (defaults to COINBASE_API_SECRET env var)
            coinbase_passphrase: Coinbase passphrase (defaults to COINBASE_PASSPHRASE env var)
            sandbox: Use Coinbase sandbox environment (for testing only)
        
        Raises:
            ValueError: If required API credentials are missing
        """
        self.api_key = coinbase_api_key or os.getenv('COINBASE_API_KEY')
        self.api_secret = coinbase_api_secret or os.getenv('COINBASE_API_SECRET')
        self.passphrase = coinbase_passphrase or os.getenv('COINBASE_PASSPHRASE')
        
        if not all([self.api_key, self.api_secret, self.passphrase]):
            raise ValueError(
                "Missing Coinbase API credentials. Set COINBASE_API_KEY, "
                "COINBASE_API_SECRET, and COINBASE_PASSPHRASE environment variables."
            )
        
        # API endpoints
        if sandbox:
            self.base_url = "https://api-sandbox.coinbase.com"
            logger.warning("Using Coinbase SANDBOX mode - no real transactions will occur")
        else:
            self.base_url = "https://api.coinbase.com"
            logger.warning("Using Coinbase PRODUCTION mode - real transactions will occur")
        
        self.sandbox = sandbox
        
    def _generate_auth_headers(
        self,
        method: str,
        request_path: str,
        body: str = ""
    ) -> Dict[str, str]:
        """
        Generate authentication headers for Coinbase API.
        
        Args:
            method: HTTP method (GET, POST, etc.)
            request_path: API endpoint path
            body: Request body (empty string for GET requests)
        
        Returns:
            Dictionary of authentication headers
        """
        timestamp = str(time.time())
        message = timestamp + method + request_path + body
        
        # Create HMAC-SHA256 signature
        signature = hmac.new(
            self.api_secret.encode('utf-8'),
            message.encode('utf-8'),
            hashlib.sha256
        ).digest()
        
        # Base64 encode signature
        import base64
        signature_b64 = base64.b64encode(signature).decode('utf-8')
        
        return {
            'CB-ACCESS-KEY': self.api_key,
            'CB-ACCESS-SIGN': signature_b64,
            'CB-ACCESS-TIMESTAMP': timestamp,
            'CB-ACCESS-PASSPHRASE': self.passphrase,
            'Content-Type': 'application/json'
        }
    
    def get_btc_price(self, currency: str = "USD") -> Dict[str, any]:
        """
        Fetch current Bitcoin price.
        
        Args:
            currency: Target currency (default: USD)
        
        Returns:
            Dictionary with price data including:
            - price: Current BTC price
            - currency: Target currency
            - timestamp: When price was fetched
        """
        try:
            url = urljoin(self.base_url, f"/v2/prices/BTC-{currency}/spot")
            headers = self._generate_auth_headers("GET", f"/v2/prices/BTC-{currency}/spot")
            
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            price = Decimal(data['data']['amount'])
            
            result = {
                'price': float(price),
                'currency': currency,
                'timestamp': datetime.now().isoformat(),
                'source': 'Coinbase'
            }
            
            logger.info(f"Current BTC price: ${price} {currency}")
            return result
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to fetch BTC price: {e}")
            raise
    
    def calculate_conversion(
        self,
        btc_amount: float
    ) -> Dict[str, any]:
        """
        Calculate USD value of Bitcoin amount.
        
        Args:
            btc_amount: Amount of Bitcoin to convert
        
        Returns:
            Dictionary with conversion details and estimated fees
        """
        price_data = self.get_btc_price()
        btc_decimal = Decimal(str(btc_amount))
        price_decimal = Decimal(str(price_data['price']))
        
        usd_value = btc_decimal * price_decimal
        
        # Estimate fees (these are approximate)
        coinbase_fee = usd_value * Decimal('0.01')  # ~1% withdrawal fee
        network_fee = Decimal('15')  # ~$15 Bitcoin network fee
        
        total_fees = coinbase_fee + network_fee
        net_usd = usd_value - total_fees
        
        return {
            'btc_amount': float(btc_decimal),
            'usd_price_per_btc': price_data['price'],
            'gross_usd_value': float(usd_value),
            'estimated_fees': {
                'coinbase_withdrawal': float(coinbase_fee),
                'network_fee': float(network_fee),
                'total_fees': float(total_fees)
            },
            'net_usd_value': float(net_usd),
            'fee_percentage': float((total_fees / usd_value) * 100) if usd_value > 0 else 0,
            'timestamp': datetime.now().isoformat(),
            'note': 'Fees are estimates and may vary. Actual fees will be shown before confirmation.'
        }
    
    def get_accounts(self) -> List[Dict]:
        """
        Retrieve all Coinbase accounts.
        
        Returns:
            List of account dictionaries with details
        """
        try:
            url = urljoin(self.base_url, "/v2/accounts")
            headers = self._generate_auth_headers("GET", "/v2/accounts")
            
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            
            accounts = response.json()['data']
            
            logger.info(f"Retrieved {len(accounts)} accounts")
            
            return accounts
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to retrieve accounts: {e}")
            raise
    
    def get_btc_account(self) -> Optional[Dict]:
        """
        Get Bitcoin account details.
        
        Returns:
            Bitcoin account dictionary or None if not found
        """
        accounts = self.get_accounts()
        
        for account in accounts:
            if account['currency']['code'] == 'BTC':
                logger.info(f"Bitcoin account balance: {account['balance']['amount']} BTC")
                return account
        
        logger.warning("No Bitcoin account found")
        return None
    
    def verify_linked_payment_methods(self) -> List[Dict]:
        """
        Retrieve all linked payment methods (bank accounts, etc.).
        
        Returns:
            List of linked payment methods
            
        Note:
            You'll need to manually link your bank account via Coinbase dashboard first
        """
        try:
            url = urljoin(self.base_url, "/v2/payment-methods")
            headers = self._generate_auth_headers("GET", "/v2/payment-methods")
            
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            
            payment_methods = response.json()['data']
            
            logger.info(f"Retrieved {len(payment_methods)} payment methods")
            
            for method in payment_methods:
                logger.info(
                    f"  - {method['type']}: {method.get('name', 'Unnamed')} "
                    f"(ID: {method['id']})"
                )
            
            return payment_methods
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to retrieve payment methods: {e}")
            raise
    
    def initiate_withdrawal(
        self,
        btc_amount: float,
        payment_method_id: str
    ) -> Dict:
        """
        Initiate Bitcoin withdrawal to linked bank account.
        
        Args:
            btc_amount: Amount of Bitcoin to withdraw
            payment_method_id: ID of linked payment method (bank account)
        
        Returns:
            Withdrawal transaction details
        
        Raises:
            ValueError: If withdrawal amount is invalid
            requests.exceptions.RequestException: If API request fails
        
        Important:
            - Withdrawals are irreversible
            - Funds may take 3-5 business days to appear in bank account
            - Bank account must be verified on both Coinbase and Cash App
        """
        if btc_amount <= 0:
            raise ValueError("Withdrawal amount must be greater than 0")
        
        try:
            # Get BTC account ID
            btc_account = self.get_btc_account()
            if not btc_account:
                raise ValueError("No Bitcoin account found")
            
            btc_account_id = btc_account['id']
            
            # Prepare withdrawal request
            url = urljoin(self.base_url, f"/v2/accounts/{btc_account_id}/withdrawals")
            
            body = json.dumps({
                'type': 'payment',
                'amount': str(btc_amount),
                'currency': 'BTC',
                'payment_method_id': payment_method_id
            })
            
            headers = self._generate_auth_headers(
                "POST",
                f"/v2/accounts/{btc_account_id}/withdrawals",
                body
            )
            
            logger.warning(
                f"⚠️  INITIATING WITHDRAWAL: {btc_amount} BTC to payment method {payment_method_id}"
            )
            
            response = requests.post(url, headers=headers, data=body, timeout=10)
            response.raise_for_status()
            
            withdrawal_data = response.json()['data']
            
            logger.info(f"Withdrawal initiated: {withdrawal_data['id']}")
            
            return {
                'status': 'initiated',
                'transaction_id': withdrawal_data['id'],
                'amount': withdrawal_data['amount'],
                'currency': withdrawal_data['currency'],
                'created_at': withdrawal_data['created_at'],
                'note': 'Funds typically appear in 3-5 business days'
            }
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Withdrawal initiation failed: {e}")
            raise
    
    def get_withdrawal_status(self, withdrawal_id: str) -> Dict:
        """
        Check status of a withdrawal.
        
        Args:
            withdrawal_id: ID of the withdrawal transaction
        
        Returns:
            Withdrawal status and details
        """
        try:
            url = urljoin(self.base_url, f"/v2/withdrawals/{withdrawal_id}")
            headers = self._generate_auth_headers("GET", f"/v2/withdrawals/{withdrawal_id}")
            
            response = requests.get(url, headers=headers, timeout=10)
            response.raise_for_status()
            
            withdrawal = response.json()['data']
            
            return {
                'id': withdrawal['id'],
                'status': withdrawal['status'],
                'amount': withdrawal['amount'],
                'currency': withdrawal['currency'],
                'created_at': withdrawal['created_at'],
                'updated_at': withdrawal['updated_at'],
                'payout_at': withdrawal.get('payout_at'),
                'fee': withdrawal.get('fee')
            }
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to retrieve withdrawal status: {e}")
            raise
    
    def get_transaction_history(self, limit: int = 10) -> List[Dict]:
        """
        Retrieve recent transactions.
        
        Args:
            limit: Maximum number of transactions to retrieve
        
        Returns:
            List of recent transactions
        """
        try:
            btc_account = self.get_btc_account()
            if not btc_account:
                return []
            
            url = urljoin(
                self.base_url,
                f"/v2/accounts/{btc_account['id']}/transactions"
            )
            headers = self._generate_auth_headers(
                "GET",
                f"/v2/accounts/{btc_account['id']}/transactions"
            )
            
            params = {'limit': limit}
            response = requests.get(url, headers=headers, params=params, timeout=10)
            response.raise_for_status()
            
            transactions = response.json()['data']
            
            logger.info(f"Retrieved {len(transactions)} transactions")
            
            return transactions
        
        except requests.exceptions.RequestException as e:
            logger.error(f"Failed to retrieve transaction history: {e}")
            raise
    
    def transfer_bitcoin_to_cashapp(
        self,
        btc_amount: float,
        payment_method_id: Optional[str] = None,
        confirm: bool = False
    ) -> Dict:
        """
        Complete Bitcoin to Cash App transfer workflow.
        
        Args:
            btc_amount: Amount of Bitcoin to transfer
            payment_method_id: ID of linked bank account (required for actual transfer)
            confirm: Set to True to actually execute the transfer
        
        Returns:
            Complete transfer summary and status
        
        IMPORTANT WORKFLOW:
            1. Verify amounts and fees
            2. Link bank account to Coinbase and Cash App
            3. Review fees (1-3% typical)
            4. Execute withdrawal
            5. Wait 3-5 business days for bank processing
            6. Transfer from bank to Cash App
        """
        logger.info("=" * 70)
        logger.info("BITCOIN TO CASH APP TRANSFER")
        logger.info("=" * 70)
        
        # Step 1: Calculate conversion
        conversion = self.calculate_conversion(btc_amount)
        
        logger.info(f"\n✓ Conversion Details:")
        logger.info(f"  Bitcoin Amount: {conversion['btc_amount']} BTC")
        logger.info(f"  Price per BTC: ${conversion['usd_price_per_btc']}")
        logger.info(f"  Gross USD Value: ${conversion['gross_usd_value']:.2f}")
        logger.info(f"  Estimated Fees: ${conversion['estimated_fees']['total_fees']:.2f}")
        logger.info(f"  Fee Percentage: {conversion['estimated_fees']:.2f}%")
        logger.info(f"  Net USD Value: ${conversion['net_usd_value']:.2f}")
        
        # Step 2: Verify account
        btc_account = self.get_btc_account()
        if not btc_account:
            return {'status': 'error', 'message': 'No Bitcoin account found'}
        
        logger.info(f"\n✓ Bitcoin Account: {btc_account['currency']['code']}")
        logger.info(f"  Available Balance: {btc_account['available']} BTC")
        
        if float(btc_account['available']) < btc_amount:
            return {
                'status': 'error',
                'message': f"Insufficient balance. Available: {btc_account['available']} BTC"
            }
        
        # Step 3: Verify payment methods
        logger.info(f"\n✓ Linked Payment Methods:")
        payment_methods = self.verify_linked_payment_methods()
        
        if not payment_methods:
            return {
                'status': 'error',
                'message': 'No linked payment methods. Link bank account via Coinbase dashboard first.'
            }
        
        # Step 4: Execute withdrawal if confirmed
        if confirm and payment_method_id:
            logger.info(f"\n⚠️  EXECUTING WITHDRAWAL")
            
            withdrawal = self.initiate_withdrawal(btc_amount, payment_method_id)
            
            return {
                'status': 'success',
                'message': 'Transfer initiated successfully',
                'conversion': conversion,
                'withdrawal': withdrawal,
                'next_steps': [
                    '1. Wait 3-5 business days for bank processing',
                    '2. Once funds arrive in your bank account, transfer to Cash App',
                    '3. Cash App transfers typically complete in 1-3 business days',
                    '4. Monitor transaction status using withdrawal ID',
                    '5. Keep records for tax reporting purposes'
                ],
                'important_notes': [
                    'This is a REAL transaction - it cannot be undone',
                    'All transactions are reported to tax authorities',
                    'Keep documentation for accounting/tax purposes',
                    'Check Coinbase and bank for updates on withdrawal status'
                ]
            }
        else:
            return {
                'status': 'preview',
                'message': 'Transfer preview (not executed). Set confirm=True to proceed.',
                'conversion': conversion,
                'warning': 'This is a PREVIEW ONLY. No funds will be transferred.',
                'required_for_execution': {
                    'payment_method_id': 'ID of linked bank account',
                    'confirm': 'Must be set to True'
                }
            }


# ============================================================================
# EXAMPLE USAGE
# ============================================================================

def main():
    """
    Example usage of Bitcoin to Cash App transfer tool.
    
    Before running:
    1. Set up environment variables:
       - COINBASE_API_KEY
       - COINBASE_API_SECRET
       - COINBASE_PASSPHRASE
    
    2. Complete KYC verification on Coinbase and Cash App
    
    3. Link your bank account to both Coinbase and Cash App
    
    4. Review all fees and terms
    """
    
    # Initialize with sandbox mode (safe for testing)
    try:
        transfer = BitcoinToCashAppTransfer(sandbox=True)
        
        # Example 1: Get current Bitcoin price
        print("\n" + "=" * 70)
        print("EXAMPLE 1: Get Bitcoin Price")
        print("=" * 70)
        price = transfer.get_btc_price()
        print(json.dumps(price, indent=2))
        
        # Example 2: Calculate conversion for 0.5 BTC
        print("\n" + "=" * 70)
        print("EXAMPLE 2: Calculate Conversion (0.5 BTC)")
        print("=" * 70)
        conversion = transfer.calculate_conversion(0.5)
        print(json.dumps(conversion, indent=2))
        
        # Example 3: Get accounts
        print("\n" + "=" * 70)
        print("EXAMPLE 3: Get Accounts")
        print("=" * 70)
        accounts = transfer.get_accounts()
        print(json.dumps(accounts, indent=2))
        
        # Example 4: Preview transfer (no execution)
        print("\n" + "=" * 70)
        print("EXAMPLE 4: Transfer Preview (Not Executed)")
        print("=" * 70)
        preview = transfer.transfer_bitcoin_to_cashapp(
            btc_amount=0.1,
            confirm=False  # Preview only
        )
        print(json.dumps(preview, indent=2))
        
        print("\n" + "=" * 70)
        print("IMPORTANT REMINDERS:")
        print("=" * 70)
        print("""
✓ KYC VERIFICATION: Both Coinbase and Cash App require identity verification
✓ BANK LINKING: Link verified bank account to both platforms
✓ FEES: Expect 1-3% in total fees (withdrawal + network)
✓ TIMING: Bank transfers take 3-5 business days
✓ TAX REPORTING: All transactions must be reported to tax authorities
✓ SECURITY: Never share API keys or passphrases
✓ CONFIRMATION: To execute real transfer, set confirm=True and provide payment_method_id
        """)
    
    except ValueError as e:
        print(f"Error: {e}")
        print("\nSet up environment variables first:")
        print("  export COINBASE_API_KEY='your_key'")
        print("  export COINBASE_API_SECRET='your_secret'")
        print("  export COINBASE_PASSPHRASE='your_passphrase'")


if __name__ == "__main__":
    main()
