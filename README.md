Workshop
# Introduction to Alith Framework and LazAI
=======
##  Telegram bot with LazAI: Your new AI Agent with easiest steps
=======
## Onchain Agents with Alith Framework: Your new AI Agent with easiest steps


Create a powerful Telegram bot that can handle token management on Metis Sepolia and respond intelligently to messages using LazAI's Alith SDK. This tutorial will guide you through the process step by step.

## Quick Start Guide

### 1. Set Up Your Environment

```bash
# Create a new project folder and navigate to it
mkdir my-telegram-bot
cd my-telegram-bot

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install required packages
pip install alith python-telegram-bot python-dotenv web3
```

### 2. Get Your API Keys

1. **Telegram Bot Token**:
   - Open Telegram and search for `@BotFather`
   - Send `/newbot` command
   - Follow prompts to name your bot
   - Save the bot token you receive

2. **OpenAI API Key**:
   - Visit [OpenAI](https://platform.openai.com)
   - Create an account or log in
   - Generate an API key

3. **Create `.env` file**:
```env
OPENAI_API_KEY=your_openai_api_key
TELEGRAM_BOT_TOKEN=your_telegram_bot_token
PRIVATE_KEY=your_wallet_private_key  # Metis Sepolia wallet private key
```

### 3. Create Your Bot

Create `tg-bot.py` with the following code:

```python
import os
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    filters,
    CallbackContext,
    CallbackQueryHandler
)
from alith import Agent
from web3 import Web3
from eth_account import Account
import json
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Metis Sepolia Configuration
METIS_SEPOLIA_CONFIG = {
    'chain_id': 59902,
    'rpc_url': 'https://sepolia.metisdevops.link',
    'explorer_url': 'https://sepolia-explorer.metisdevops.link',
    'name': 'Metis Sepolia'
}

# Initialize Web3
w3 = Web3(Web3.HTTPProvider(METIS_SEPOLIA_CONFIG['rpc_url']))

# ERC20 ABI
ERC20_ABI = ''' ERC20 ABIT key'''

# ERC721 ABI (Basic NFT)
ERC721_ABI = '''ERC721 ABI key'''

# ERC1155 ABI (Multi Token)
ERC1155_ABI = '''ERC1155 key'''

# Smart Contract Bytecode (placeholders - you would need the actual deployment bytecode)
ERC20_BYTECODE = "0x608060405234801561001057600080fd5b506040516108..."  # Replace with actual bytecode
ERC721_BYTECODE = "0x608060405234801561001057600080fd5b506040516108..."  # Replace with actual bytecode
ERC1155_BYTECODE = "0x608060405234801561001057600080fd5b506040516109..."  # Replace with actual bytecode

# Initialize Alith Agent
agent = Agent(
    name="Telegram Bot Agent",
    model="gpt-4",
    preamble="""You are an advanced AI assistant built by [Alith](https://github.com/0xLazAI/alith).""",
)

# Initialize Telegram Bot
bot_token = os.getenv("TELEGRAM_BOT_TOKEN")
app = Application.builder().token(bot_token).build()

# Store user state for multi-step deployments
user_states = {}

def is_valid_address(address: str) -> bool:
    """Validate Ethereum address format"""
    return address and len(address) == 42 and address.startswith('0x') and all(c in '0123456789abcdefABCDEF' for c in address[2:])

async def check_balance(contract_address: str, wallet_address: str) -> str:
    try:
        # Validate addresses
        if not is_valid_address(contract_address) or not is_valid_address(wallet_address):
            return "❌ Invalid address format. Addresses should be 42 characters long and start with '0x'"

        # Create contract instance
        contract = w3.eth.contract(
            address=Web3.to_checksum_address(contract_address),
            abi=json.loads(ERC20_ABI)
        )
        
        # Get token info
        try:
            symbol = contract.functions.symbol().call()
            decimals = contract.functions.decimals().call()
            name = contract.functions.name().call()
            total_supply = contract.functions.totalSupply().call()
        except Exception as e:
            return f"❌ Error reading token information: {str(e)}"
        
        # Get balance
        balance = contract.functions.balanceOf(
            Web3.to_checksum_address(wallet_address)
        ).call()
                
        return (
            f"💰 Token Balance Report\n\n"
            f"Token Name: {name}\n"
            f"Symbol: {symbol}\n"
            f"Total Supply: {total_supply} {symbol}\n\n"
            f"Wallet: {wallet_address}\n"
            f"Balance: {balance} {symbol}\n\n"
            f"🔍 View on Explorer:\n"
            f"Token: {METIS_SEPOLIA_CONFIG['explorer_url']}/token/{contract_address}\n"
            f"Wallet: {METIS_SEPOLIA_CONFIG['explorer_url']}/address/{wallet_address}"
        )
    
    except Exception as e:
        return f"❌ Error checking balance: {str(e)}"

def show_token_deployment_options(chat_id: int) -> InlineKeyboardMarkup:
    """Create inline keyboard with token deployment options"""
    keyboard = [
        [
            InlineKeyboardButton("Deploy ERC20 Token", callback_data="deploy_erc20"),
            InlineKeyboardButton("Deploy ERC721 NFT", callback_data="deploy_erc721")
        ],
        [
            InlineKeyboardButton("Deploy ERC1155 Multi-Token", callback_data="deploy_erc1155"),
            InlineKeyboardButton("Cancel", callback_data="cancel_deploy")
        ]
    ]
    return InlineKeyboardMarkup(keyboard)

async def handle_callback_query(update: Update, context: CallbackContext) -> None:
    """Handle callback queries from inline keyboards"""
    query = update.callback_query
    chat_id = update.effective_chat.id
    
    # Always answer the callback query to remove the loading indicator
    await query.answer()
    
    if query.data == "cancel_deploy":
        await query.edit_message_text("Token deployment cancelled.")
        if chat_id in user_states:
            del user_states[chat_id]
        return
    
    if query.data.startswith("deploy_erc"):
        token_type = query.data.split("_")[1].upper()
        user_states[chat_id] = {
            "deploying": True,
            "token_type": token_type,
            "step": "get_wallet"
        }
        
        await query.edit_message_text(
            f"Let's deploy your {token_type} token!\n\n"
            f"Please provide your wallet address that will own the token:"
        )
        return
    
    # Handle other callback queries if needed

async def deploy_token(chat_id: int, token_type: str, params: dict) -> str:
    """Deploy a token based on type and parameters"""
    try:
        # Get private key from environment or user input (securely)
        private_key = os.getenv("DEPLOYER_PRIVATE_KEY")
        if not private_key:
            return "❌ Deployer private key not found in environment variables"
        
        # Create account from private key
        account = Account.from_key(private_key)
        
        # Get transaction parameters
        tx_params = {
            'from': account.address,
            'nonce': w3.eth.get_transaction_count(account.address),
            'gasPrice': w3.eth.gas_price,
            'chainId': METIS_SEPOLIA_CONFIG['chain_id']
        }
        
        contract_data = None
        constructor_args = None
        
        if token_type == "ERC20":
            contract = w3.eth.contract(abi=json.loads(ERC20_ABI), bytecode=ERC20_BYTECODE)
            constructor_args = [
                params['name'],
                params['symbol'],
                int(float(params['supply'])),  # Convert to smallest units
                Web3.to_checksum_address(params['owner'])
            ]
        elif token_type == "ERC721":
            contract = w3.eth.contract(abi=json.loads(ERC721_ABI), bytecode=ERC721_BYTECODE)
            constructor_args = [
                params['name'],
                params['symbol'],
                params['base_uri']
            ]
        elif token_type == "ERC1155":
            contract = w3.eth.contract(abi=json.loads(ERC1155_ABI), bytecode=ERC1155_BYTECODE)
            constructor_args = [params['uri']]
        
        # Build constructor transaction
        contract_tx = contract.constructor(*constructor_args).build_transaction(tx_params)
        
        # Estimate gas
        gas_estimate = w3.eth.estimate_gas(contract_tx)
        contract_tx['gas'] = gas_estimate
        
        # Sign transaction
        signed_tx = w3.eth.account.sign_transaction(contract_tx, private_key)
        
        # Send transaction - Using the correct attribute for Web3.py
        tx_hash = w3.eth.send_raw_transaction(signed_tx.raw_transaction)
        
        # Wait for transaction receipt
        tx_receipt = w3.eth.wait_for_transaction_receipt(tx_hash)
        
        contract_address = tx_receipt.contractAddress
        
        return (
            f"✅ {token_type} Token Deployed Successfully!\n\n"
            f"Contract Address: {contract_address}\n"
            f"Transaction Hash: {tx_hash.hex()}\n\n"
            f"🔍 View on Explorer:\n"
            f"{METIS_SEPOLIA_CONFIG['explorer_url']}/address/{contract_address}\n"
            f"{METIS_SEPOLIA_CONFIG['explorer_url']}/tx/{tx_hash.hex()}"
        )
    
    except Exception as e:
        return f"❌ Error deploying {token_type} token: {str(e)}"
        
async def process_deployment_steps(update: Update, context: CallbackContext, chat_id: int, user_state: dict) -> None:
    """Process token deployment steps based on user state"""
    token_type = user_state["token_type"]
    step = user_state["step"]
    message_text = update.message.text.strip()
    
    if step == "get_wallet":
        if not is_valid_address(message_text):
            await context.bot.send_message(
                chat_id=chat_id,
                text="❌ Invalid wallet address. Please provide a valid Ethereum address starting with 0x."
            )
            return
        
        user_state["owner"] = message_text
        user_state["step"] = "get_name"
        
        await context.bot.send_message(
            chat_id=chat_id,
            text=f"Great! Now enter a name for your {token_type} token:"
        )
    
    elif step == "get_name":
        user_state["name"] = message_text
        user_state["step"] = "get_symbol"
        
        await context.bot.send_message(
            chat_id=chat_id,
            text="Now enter a symbol for your token (e.g. BTC, ETH):"
        )
    
    elif step == "get_symbol":
        user_state["symbol"] = message_text
        
        if token_type == "ERC20":
            user_state["step"] = "get_supply"
            await context.bot.send_message(
                chat_id=chat_id,
                text="Enter the initial supply for your token (e.g. 1000000):"
            )
        elif token_type == "ERC721":
            user_state["step"] = "get_base_uri"
            await context.bot.send_message(
                chat_id=chat_id,
                text="Enter the base URI for your NFT metadata (e.g. https://example.com/metadata/):"
            )
        elif token_type == "ERC1155":
            user_state["step"] = "get_uri"
            await context.bot.send_message(
                chat_id=chat_id,
                text="Enter the URI for your multi-token metadata (e.g. https://example.com/metadata/{id}.json):"
            )
    
    elif step == "get_supply":
        try:
            supply = float(message_text)
            user_state["supply"] = message_text
            user_state["step"] = "confirm"
            
            # Show confirmation
            await context.bot.send_message(
                chat_id=chat_id,
                text=(
                    f"📋 Deployment Summary (ERC20):\n\n"
                    f"Name: {user_state['name']}\n"
                    f"Symbol: {user_state['symbol']}\n"
                    f"Supply: {user_state['supply']}\n"
                    f"Owner: {user_state['owner']}\n\n"
                    f"Type 'confirm' to deploy or 'cancel' to abort."
                )
            )
        except ValueError:
            await context.bot.send_message(
                chat_id=chat_id,
                text="❌ Invalid supply. Please enter a valid number."
            )
    
    elif step == "get_base_uri" or step == "get_uri":
        uri_key = "base_uri" if step == "get_base_uri" else "uri"
        user_state[uri_key] = message_text
        user_state["step"] = "confirm"
        
        # Show confirmation
        await context.bot.send_message(
            chat_id=chat_id,
            text=(
                f"📋 Deployment Summary ({token_type}):\n\n"
                f"Name: {user_state['name']}\n"
                f"Symbol: {user_state['symbol']}\n"
                f"{uri_key.replace('_', ' ').title()}: {user_state[uri_key]}\n"
                f"Owner: {user_state['owner']}\n\n"
                f"Type 'confirm' to deploy or 'cancel' to abort."
            )
        )
    
    elif step == "confirm":
        if message_text.lower() == "confirm":
            # Send initial status
            status_message = await context.bot.send_message(
                chat_id=chat_id,
                text=f"🔄 Deploying your {token_type} token... Please wait."
            )
            
            # Deploy token
            params = {
                "owner": user_state["owner"],
                "name": user_state["name"],
                "symbol": user_state["symbol"]
            }
            
            if "supply" in user_state:
                params["supply"] = user_state["supply"]
            if "base_uri" in user_state:
                params["base_uri"] = user_state["base_uri"]
            if "uri" in user_state:
                params["uri"] = user_state["uri"]
            
            result = await deploy_token(chat_id, token_type, params)
            
            # Update with result
            await context.bot.edit_message_text(
                chat_id=chat_id,
                message_id=status_message.message_id,
                text=result
            )
            
            # Clear user state
            del user_states[chat_id]
        
        elif message_text.lower() == "cancel":
            await context.bot.send_message(
                chat_id=chat_id,
                text="Token deployment cancelled."
            )
            del user_states[chat_id]
        
        else:
            await context.bot.send_message(
                chat_id=chat_id,
                text="Please type 'confirm' to proceed with deployment or 'cancel' to abort."
            )

async def handle_message(update: Update, context: CallbackContext) -> None:
    chat_id = update.effective_chat.id
    message_text = update.message.text.lower()
    
    # Check if user is in deployment flow
    if chat_id in user_states and user_states[chat_id].get("deploying"):
        await process_deployment_steps(update, context, chat_id, user_states[chat_id])
        return
    
    # Handle greetings
    if message_text in ["hi", "hello", "hey", "hola", "greetings"]:
        await context.bot.send_message(
            chat_id=chat_id,
            text=(
                "👋 Hello! Would you like to deploy a token on Metis Sepolia?"
            ),
            reply_markup=show_token_deployment_options(chat_id)
        )
        return
    
    # Handle balance command
    if message_text.startswith('/balance'):
        await check_balance_command(update, context)
    elif "check balance" in message_text.lower() or "show balance" in message_text.lower():
        try:
            # Find addresses in the message
            words = message_text.split()
            addresses = [word for word in words if is_valid_address(word)]
            
            if len(addresses) < 2:
                await context.bot.send_message(
                    chat_id=chat_id,
                    text=(
                        "Please provide both contract and wallet addresses.\n"
                        "Examples:\n"
                        "1. Check balance for contract 0x123... wallet 0x456...\n"
                        "2. /balance 0x123... 0x456..."
                    )
                )
                return
            
            contract_address = addresses[0]
            wallet_address = addresses[1]
            
            # Send initial status
            status_message = await context.bot.send_message(
                chat_id=chat_id,
                text="🔍 Checking token balance..."
            )
            
            # Check balance
            result = await check_balance(contract_address, wallet_address)
            
            # Update with result
            await context.bot.edit_message_text(
                chat_id=chat_id,
                message_id=status_message.message_id,
                text=result
            )
            
        except Exception as e:
            await context.bot.send_message(
                chat_id=chat_id,
                text=f"❌ Error: {str(e)}\n\nPlease use the format:\nCheck balance for contract 0x123... wallet 0x456..."
            )
    else:
        # Handle other messages with AI agent
        response = agent.prompt(update.message.text)
        await context.bot.send_message(chat_id=chat_id, text=response)

async def check_balance_command(update: Update, context: CallbackContext):
    try:
        params = update.message.text.split()
        if len(params) != 3:
            await context.bot.send_message(
                chat_id=update.effective_chat.id,
                text=(
                    "Please provide contract and wallet address:\n"
                    "/balance <contract_address> <wallet_address>"
                )
            )
            return
        
        _, contract_address, wallet_address = params
        
        # Send initial status
        status_message = await context.bot.send_message(
            chat_id=update.effective_chat.id,
            text="🔍 Checking token balance..."
        )
        
        # Check balance
        result = await check_balance(contract_address, wallet_address)
        
        # Update with result
        await context.bot.edit_message_text(
            chat_id=update.effective_chat.id,
            message_id=status_message.message_id,
            text=result
        )
        
    except Exception as e:
        await context.bot.send_message(
            chat_id=update.effective_chat.id,
            text=f"❌ Error: {str(e)}"
        )

# Add handlers
app.add_handler(CommandHandler("balance", handle_message))
app.add_handler(CallbackQueryHandler(handle_callback_query))
app.add_handler(MessageHandler(filters.TEXT & (~filters.COMMAND), handle_message))

# Start bot
if __name__ == "__main__":
    print(f"🤖 Bot starting on {METIS_SEPOLIA_CONFIG['name']}...")
    print("Available commands:")
    print("1. /balance <contract_address> <wallet_address>")
    print("2. Natural language: 'Check balance for contract 0x... wallet 0x...'")
    print("3. Say 'hi' or 'hello' to get token deployment options")
    app.run_polling()
```

### 4. Add Token Management Features
Check the tg-bot.py file for full code reference.

Add these functions to your bot for token management:

```python
async def check_balance(contract_address: str, wallet_address: str) -> dict:
    """Check token balance"""
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(contract_address),
        abi=ERC20_ABI
    )
    balance = contract.functions.balanceOf(
        Web3.to_checksum_address(wallet_address)
    ).call()
    return balance

async def transfer_token(contract_address: str, to_address: str, amount: float) -> str:
    """Transfer tokens"""
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(contract_address),
        abi=ERC20_ABI
    )
    
    # Build transaction
    transfer_txn = contract.functions.transfer(
        Web3.to_checksum_address(to_address),
        amount
    ).build_transaction({
        'chainId': 59902,  # Metis Sepolia
        'gas': 100000,
        'nonce': w3.eth.get_transaction_count(WALLET_ADDRESS),
    })
    
    # Sign and send
    signed_txn = w3.eth.account.sign_transaction(transfer_txn, PRIVATE_KEY)
    tx_hash = w3.eth.send_raw_transaction(signed_txn.raw_transaction)
    
    return tx_hash.hex()
```

### 5. Run Your Bot

```bash
python tg-bot.py
```

## Available Commands

Once your bot is running, you can use these commands in Telegram:

```
/balance <contract_address> <wallet_address>
Check token balance

/transfer <contract_address> <to_address> <amount>
Transfer tokens

Or use natural language:
"Check balance for contract 0x... wallet 0x..."
"Transfer 100 tokens from contract 0x... to 0x..."
```
## Sample Output
When you check a token balance, you'll see a response like this:



## Features

- AI-powered chat responses
- Token balance checking
- Token transfers
- Natural language command processing

## Deployment Tips

1. **Local Testing**:
   - Keep your `.env` file secure
   - Test all features before deployment

2. **Production Deployment**:
   - Deploy to AWS Lambda, Heroku, or your preferred platform
   - Set environment variables in your deployment platform
   - Monitor bot performance and errors

## Troubleshooting

Common issues and solutions:

1. **Bot not responding**:
   - Check if your bot token is correct
   - Ensure your bot is running
   - Verify internet connection

2. **Token operations failing**:
   - Confirm wallet has enough balance
   - Verify contract addresses are correct
   - Check Metis Sepolia network status

## Need Help?

- Check [python-telegram-bot docs](https://python-telegram-bot.readthedocs.io/)
- Visit [LazAI GitHub](https://github.com/0xLazAI/alith) for SDK updates
- Join Telegram development community

