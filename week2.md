## Exercise 1

```rust
use std::{fmt::Display};

struct Calculator<X, Y> {x: X, y: Y}

trait AdditiveOperations {
    type Output;

    fn add(&self) -> Self::Output;

    fn sub(&self) -> Self::Output;
}

impl<X, Y, O> AdditiveOperations for Calculator<X, Y>
where
    Y: Copy,
    X: Copy + std::ops::Add<Y, Output = O> + std::ops::Sub<Y, Output = O> {
    type Output = O;

    fn add(&self) -> Self::Output {
        self.x + self.y
    }

    fn sub(&self) -> Self::Output {
        self.x - self.y
    }
}

trait MultiplicativeOperations {
    type Output;

    fn mul(&self) -> Self::Output;

    fn div(&self) -> Self::Output;
}

impl<X, Y, O> MultiplicativeOperations for Calculator<X, Y>
where
    Y: Copy,
    X: Copy + std::ops::Mul<Y, Output = O> + std::ops::Div<Y, Output = O> {
    type Output = O;
    
    fn mul(&self) -> Self::Output {
        self.x * self.y
    }
    
    fn div(&self) -> Self::Output {
        self.x / self.y
    }
}

trait BinaryOperations {
    type Output;

    fn and(&self) -> Self::Output;

    fn or(&self) -> Self::Output;

    fn xor(&self) -> Self::Output;
}

impl<X, Y, O> BinaryOperations for Calculator<X, Y>
where
    Y: Copy,
    X: Copy + std::ops::BitAnd<Y, Output = O> + std::ops::BitOr<Y, Output = O> + std::ops::BitXor<Y, Output = O> {
    type Output = O;

    fn and(&self) -> Self::Output {
        self.x & self.y
    }

    fn or(&self) -> Self::Output {
        self.x | self.y
    }

    fn xor(&self) -> Self::Output {
        self.x ^ self.y
    }
}

impl<X, Y> std::fmt::Display for Calculator<X, Y>
where
    X: std::fmt::Display,
    Y: std::fmt::Display,
    Calculator<X, Y>: AdditiveOperations<Output: Display>,
    Calculator<X, Y>: MultiplicativeOperations<Output: Display>,
    Calculator<X, Y>: BinaryOperations<Output: Display> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Calculator({}, {})\n", self.x, self.y)?;
        write!(f, "Addition: {}\n", self.add())?;
        write!(f, "Subtraction: {}\n", self.sub())?;
        write!(f, "Multiplication: {}\n", self.mul())?;
        write!(f, "Division: {}\n", self.div())?;
        write!(f, "AND: {}\n", self.and())?;
        write!(f, "OR: {}\n", self.or())?;
        write!(f, "XOR: {}\n", self.xor())
    }
}

fn main() {
    let calculator = Calculator{
        x: 1,
        y: 2
    };
    println!("calculator: {}", calculator); 
    // Does not compile: cannot divide integer by float
    /*let calculator2 = Calculator{
        x: 1,
        y: 2.5
    };
    println!("calculator: {}", calculator2); */
    // let calculator3: Calculator<i64, i32> = Calculator{
    //     x: 1,
    //     y: 2
    // };
    // println!("calculator: {}", calculator3);

}
```

## Exercise 2

The contract stores a vector of messages on-chain. Users can send a message - appending it to the vector - by calling `Contract.addMessage`.
If (and only if) they call this function with value of at least `POINT_ONE` then their message will contain `premium = true`. The `sender` field
of the message is always the caller of `addMessage` (could be another contract or an EOA). The non-payable `get_messages` and `total_messages` are read-only
functions (as they do an immutable borrow of the `Contract`), which allow for reading a contiguous sequence of messages and getting the total number of messages.

```rust
use near_sdk::borsh::{BorshDeserialize, BorshSerialize};
use near_sdk::json_types::U64;
use near_sdk::serde::Serialize;
use near_sdk::store::Vector;
use near_sdk::{env, near_bindgen, AccountId, NearToken};

const POINT_ONE: NearToken = NearToken::from_millinear(100);

#[derive(BorshDeserialize, BorshSerialize, Serialize)] // Auto-generates implementations for these traits
#[serde(crate = "near_sdk::serde")]
#[borsh(crate = "near_sdk::borsh")] // I assume this introduces special serialization logic for Near-specific types?
pub struct PostedMessage {
    pub premium: bool,
    pub sender: AccountId,
    pub text: String,
}

#[near_bindgen] // Auto-generates code for calling the contract from off-chain?
#[derive(BorshDeserialize, BorshSerialize)] // Auto-generates implementations for these traits
#[borsh(crate = "near_sdk::borsh")]
pub struct Contract {
    messages: Vector<PostedMessage>, // The stored state of the contract
}

impl Default for Contract {
    fn default() -> Self { // returns a default instance of Contract
        Self {
            messages: Vector::new(b"m"), // The new zero-length vector will be stored at a location determined by the prefix "m" --
                                         // This prevents collisions with other collections stored on the contract
        }
    }
}

#[near_bindgen]
impl Contract {
    #[payable]
    pub fn add_message(&mut self, text: String) {
        let premium = env::attached_deposit() >= POINT_ONE;
        let sender = env::predecessor_account_id();

        let message = PostedMessage {
            premium,
            sender,
            text,
        };

        self.messages.push(message);
    }

    pub fn get_messages(&self, from_index: Option<U64>, limit: Option<U64>) -> Vec<&PostedMessage> {
        let from = u64::from(from_index.unwrap_or(U64(0)));
        let limit = u64::from(limit.unwrap_or(U64(10)));

        self.messages
            .iter()
            .skip(from as usize)
            .take(limit as usize)
            .collect()
    }

    pub fn total_messages(&self) -> u32 {
        self.messages.len()
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_message() {
        let mut contract = Contract::default();
        contract.add_message("A message".to_string());

        let posted_message = &contract.get_messages(None, None)[0];
        assert_eq!(posted_message.premium, false);
        assert_eq!(posted_message.text, "A message".to_string());
    }

    #[test]
    fn iters_messages() {
        let mut contract = Contract::default();
        contract.add_message("1st message".to_string());
        contract.add_message("2nd message".to_string());
        contract.add_message("3rd message".to_string());

        let total = &contract.total_messages();
        assert!(*total == 3);

        let last_message = &contract.get_messages(Some(U64::from(1)), Some(U64::from(2)))[1];
        assert_eq!(last_message.premium, false);
        assert_eq!(last_message.text, "3rd message".to_string());
    }
}
```

## Exercise 3

The contract allows users to deposit ERC20 tokens using the `ISimpleVault` interface.
The `SimpleVault` defines its storage to contain the `token` (used for deposits/withdrawals), `total_supply` (amount of currently issued shares) and a
`balance_of` mapping (tracks number of shares per address).

```rust
use starknet::ContractAddress;

// Interface for ERC20 tokens
#[starknet::interface] // generates IERC20Dispatcher (a struct) and IERC20DispatcherTrait
pub trait IERC20<TContractState> { // type parameter TContractState defines the type of the state stored by the contract
    // Read-only functions (because they use `self: @TContractState` i.e. an immutable snapshot of the state)
    fn get_name(self: @TContractState) -> felt252;
    fn get_symbol(self: @TContractState) -> felt252;
    fn get_decimals(self: @TContractState) -> u8;
    fn get_total_supply(self: @TContractState) -> felt252;
    fn balance_of(self: @TContractState, account: ContractAddress) -> felt252;
    fn allowance(
        self: @TContractState, owner: ContractAddress, spender: ContractAddress
    ) -> felt252;

    // State changing functions (because they use `ref self: TContractState` i.e. a mutable reference to the state)
    fn transfer(ref self: TContractState, recipient: ContractAddress, amount: felt252);
    fn transfer_from(
        ref self: TContractState,
        sender: ContractAddress,
        recipient: ContractAddress,
        amount: felt252
    );
    fn approve(ref self: TContractState, spender: ContractAddress, amount: felt252);
    fn increase_allowance(ref self: TContractState, spender: ContractAddress, added_value: felt252);
    fn decrease_allowance(
        ref self: TContractState, spender: ContractAddress, subtracted_value: felt252
    );
}

// Interface for vault contracts
#[starknet::interface]
pub trait ISimpleVault<TContractState> { // type parameter TContractState defines the type of the state stored by the contract
    fn deposit(ref self: TContractState, amount: u256);
    fn withdraw(ref self: TContractState, shares: u256);
    fn user_balance_of(ref self: TContractState, account: ContractAddress) -> u256;
    fn contract_total_supply(ref self: TContractState) -> u256;
}

#[starknet::contract] // Macro to generate code which makes this module into a contract
                      // e.g. generates ContractState struct
pub mod SimpleVault {
    use super::{IERC20Dispatcher, IERC20DispatcherTrait}; // brings these items into namespace
    use starknet::{ContractAddress, get_caller_address, get_contract_address};

    #[storage]
    struct Storage { // Data stored on the contract
        token: IERC20Dispatcher,
        total_supply: u256,
        balance_of: LegacyMap<ContractAddress, u256> // QUESTION: why `ContractAddress` and not other addresses?
    }

    #[constructor]
    fn constructor(ref self: ContractState, token: ContractAddress) {
        self.token.write(IERC20Dispatcher { contract_address: token });
    }

    #[generate_trait]
    impl PrivateFunctions of PrivateFunctionsTrait {
        fn _mint(ref self: ContractState, to: ContractAddress, shares: u256) {
            self.total_supply.write(self.total_supply.read() + shares);
            self.balance_of.write(to, self.balance_of.read(to) + shares);
        }

        fn _burn(ref self: ContractState, from: ContractAddress, shares: u256) {
            self.total_supply.write(self.total_supply.read() - shares);
            self.balance_of.write(from, self.balance_of.read(from) - shares);
        }
        
    }

    #[abi(embed_v0)]
    impl SimpleVault of super::ISimpleVault<ContractState> { // named impl of ISimpleVault for the ContractState struct

        fn user_balance_of(ref self: ContractState, account: ContractAddress) -> u256 {
            self.balance_of.read(account)
        }

        fn contract_total_supply(ref self: ContractState) -> u256 {
            self.total_supply.read()
        }


        fn deposit(ref self: ContractState, amount: u256){
            let caller = get_caller_address();
            let this = get_contract_address();

            let mut shares = 0;
            if self.total_supply.read() == 0 {
                shares = amount;
            } else {
                let balance: u256 = self.token.read().balance_of(this).try_into() // try_into: converts felt252 -> Option<u256>
                .unwrap(); // Fails if Option is None
                shares = (amount * self.total_supply.read()) / balance;
            }

            PrivateFunctions::_mint(ref self, caller, shares); // mutate contract state

            let amount_felt252: felt252 = amount.low.into(); // Converts lower 128 bits of `amount` into felt252
            self.token.read().transfer_from(caller, this, amount_felt252);
        }

        fn withdraw(ref self: ContractState, shares: u256) {
            let caller = get_caller_address();
            let this = get_contract_address();

            let balance = self.user_balance_of(this);
            let amount = (shares * balance) / self.total_supply.read();
            PrivateFunctions::_burn(ref self, caller, shares);
            let amount_felt252: felt252 = amount.low.into();
            self.token.read().transfer(caller, amount_felt252);
        }
    }
}
```