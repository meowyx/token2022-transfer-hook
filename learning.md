# Learning Path: Anchor + Rust for Solana Token2022 Transfer Hook

A step-by-step guide to understand every concept used in this project.

---

## Phase 1: Rust Fundamentals

Read these in order. You don't need to master everything — focus on the sections listed.

### 1.1 Ownership, Borrowing & References
- **Why:** Every Solana account access uses borrows (`&`, `&mut`). The transfer hook's `try_borrow_mut_data()` won't make sense without this.
- **Read:** [The Rust Book — Ch 4: Ownership](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
- **Key sections:** Ownership rules, references & borrowing, the slice type

### 1.2 Structs and Methods
- **Why:** Every account in this project is a struct (`WhitelistEntry`, `TokenFactory`). Methods are defined in `impl` blocks.
- **Read:** [The Rust Book — Ch 5: Structs](https://doc.rust-lang.org/book/ch05-00-structs.html)
- **Key sections:** Defining structs, method syntax with `impl`, `&self` vs `&mut self`

### 1.3 Enums and Pattern Matching
- **Why:** Anchor's `Result<()>` is an enum. Error handling uses `?` operator extensively.
- **Read:** [The Rust Book — Ch 6: Enums](https://doc.rust-lang.org/book/ch06-00-enums.html)
- **Key sections:** `Option<T>`, `Result<T, E>`, the `?` operator

### 1.4 Modules and Project Structure
- **Why:** This project uses `mod instructions; mod state; pub use ...` to organize code across files.
- **Read:** [The Rust Book — Ch 7: Packages and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
- **Key sections:** `mod`, `pub`, `use`, separating modules into files
- **Then look at:** `src/instructions/mod.rs` and `src/state/mod.rs` in this project

### 1.5 Generics and Traits
- **Why:** Anchor uses generics everywhere — `Context<'info>`, `Account<'info, WhitelistEntry>`, `InterfaceAccount<'info, Mint>`.
- **Read:** [The Rust Book — Ch 10: Generics and Traits](https://doc.rust-lang.org/book/ch10-00-generics.html)
- **Key sections:** Generic types in structs, trait bounds, lifetime annotations (`'info`)

### 1.6 Derive Macros and Attributes
- **Why:** `#[account]`, `#[derive(Accounts)]`, `#[instruction(...)]` are all macros that generate code for you.
- **Read:** [The Rust Book — Ch 19.5: Macros](https://doc.rust-lang.org/book/ch20-05-macros.html)
- **Key concept:** Derive macros auto-implement traits. `#[derive(Accounts)]` generates account validation logic.

---

## Phase 2: Solana Core Concepts

### 2.1 Solana Account Model
- **Why:** Everything on Solana is an account. Understanding this is prerequisite to everything else.
- **Read:** [Solana Docs — Accounts](https://solana.com/docs/core/accounts)
- **Key concepts:**
  - Accounts store data + lamports + owner program
  - Programs are stateless; state lives in accounts
  - Rent and rent-exemption

### 2.2 Programs and Instructions
- **Why:** Your program exposes instructions (`add_to_whitelist`, `init_mint`, etc). Clients call them via transactions.
- **Read:** [Solana Docs — Programs](https://solana.com/docs/core/programs)
- **Key concepts:**
  - Instructions = function calls to on-chain programs
  - Each instruction specifies accounts it reads/writes
  - Transactions bundle multiple instructions

### 2.3 Program Derived Addresses (PDAs)
- **Why:** This entire project revolves around PDAs — whitelist entries, extra account metas, token accounts.
- **Read:** [Solana Docs — PDAs](https://solana.com/docs/core/pda)
- **Key concepts:**
  - PDAs are deterministic addresses derived from seeds + program ID
  - `findProgramAddress(seeds, programId)` — off-chain derivation
  - PDAs have no private key — only the program can sign for them
  - **In this project:** `seeds = [b"whitelist", user.as_ref()]` creates one PDA per user

### 2.4 Cross-Program Invocations (CPI)
- **Why:** The transfer hook is invoked via CPI from the Token2022 program.
- **Read:** [Solana Docs — CPIs](https://solana.com/docs/core/cpi)
- **Key concept:** One program calling another program's instruction

---

## Phase 3: Anchor Framework

### 3.1 Anchor Basics
- **Why:** Anchor abstracts away raw Solana boilerplate. This project is 100% Anchor.
- **Read:** [Anchor Book — Getting Started](https://www.anchor-lang.com/docs/getting-started)
- **Key concepts:**
  - `declare_id!` — sets the program's on-chain address
  - `#[program]` — marks the module containing instruction handlers
  - `Context<T>` — gives access to accounts and bumps

### 3.2 Account Constraints (the most important part)
- **Why:** Constraints like `init`, `seeds`, `close`, `payer`, `space` are used on every account in this project.
- **Read:** [Anchor Book — Account Constraints](https://www.anchor-lang.com/docs/account-constraints)
- **Map to this project:**

| Constraint | Where used | What it does |
|---|---|---|
| `init` | `AddToWhitelist`, `TokenFactory` | Creates and initializes an account |
| `seeds + bump` | Whitelist PDAs, ExtraAccountMeta | Derives and validates PDA address |
| `close = admin` | `RemoveFromWhitelist` | Closes account, sends lamports to admin |
| `payer = admin` | `AddToWhitelist` | Who pays rent for new account |
| `space = 8 + 1` | `AddToWhitelist` | Account size: 8 (discriminator) + 1 (bump) |
| `mint::decimals` | `TokenFactory` | Initializes mint with 9 decimals |
| `mint::authority` | `TokenFactory` | Sets mint authority |
| `token::mint` | `TransferHook` | Validates token account belongs to mint |
| `token::authority` | `TransferHook` | Validates token account authority |
| `extensions::transfer_hook::*` | `TokenFactory` | Configures transfer hook extension on mint |

### 3.3 The `#[instruction(...)]` Attribute
- **Why:** Used in `AddToWhitelist` and `RemoveFromWhitelist` to access the `user: Pubkey` argument in seed derivation.
- **Read:** [Anchor Book — Account Constraints](https://www.anchor-lang.com/docs/account-constraints) (search for "instruction")
- **In this project:** `#[instruction(user: Pubkey)]` lets you use `user.as_ref()` in `seeds = [b"whitelist", user.as_ref()]`

### 3.4 Account Types
- **Why:** Knowing when to use `Account`, `InterfaceAccount`, `UncheckedAccount`, `Signer`, `Program` matters.
- **Read:** [Anchor Book — Account Types](https://www.anchor-lang.com/docs/account-types)

| Type | When to use | Example in project |
|---|---|---|
| `Signer<'info>` | Must sign the transaction | `admin`, `user` |
| `Account<'info, T>` | Anchor-owned, deserialized | `whitelist_entry` |
| `InterfaceAccount<'info, T>` | SPL program accounts (Mint, TokenAccount) | `mint`, `source_token` |
| `UncheckedAccount<'info>` | No validation needed / external program | `owner`, `extra_account_meta_list` |
| `Program<'info, T>` | A program account | `system_program` |
| `Interface<'info, T>` | SPL program interface | `token_program` |

### 3.5 The #[account] Macro
- **Why:** `#[account]` on `WhitelistEntry` auto-derives serialization, discriminator, and owner checks.
- **Key insight:** Anchor uses an 8-byte discriminator (hash of the struct name) as the first 8 bytes of account data. This is why `space = 8 + 1` — 8 for discriminator, 1 for the `bump` field.

---

## Phase 4: Token2022 and Transfer Hook

### 4.1 Token Extensions (Token2022)
- **Why:** This project uses Token2022's transfer hook extension.
- **Read:** [Solana Docs — Token Extensions](https://solana.com/docs/programs/token-2022)
- **Key concept:** Token2022 extends SPL Token with optional extensions (transfer hook, transfer fee, confidential transfers, etc.)

### 4.2 Transfer Hook Extension
- **Why:** This is what the whole project implements.
- **Read:** [Solana Docs — Transfer Hook](https://solana.com/docs/programs/token-2022/extensions/transfer-hook)
- **How it works:**
  1. A mint is created with the transfer hook extension, pointing to your program
  2. On every `transferChecked`, Token2022 CPIs into your program's `transfer_hook` instruction
  3. Your program can allow or deny the transfer

### 4.3 ExtraAccountMetaList
- **Why:** The transfer hook needs extra accounts (like the whitelist PDA) beyond the standard transfer accounts. ExtraAccountMetaList tells Token2022 which extra accounts to pass.
- **Read:** [spl-transfer-hook-interface docs](https://docs.rs/spl-transfer-hook-interface/latest/spl_transfer_hook_interface/)
- **In this project:**
  - `init_extra_account_meta.rs` stores a list saying "resolve a PDA from seeds `[b'whitelist', owner_key]`"
  - At transfer time, Token2022 reads this list and includes the resolved PDA
  - `Seed::Literal { bytes: b"whitelist" }` — a fixed seed
  - `Seed::AccountKey { index: 3 }` — dynamically uses account at index 3 (the owner) as a seed

### 4.4 The Transfer Hook Account Ordering
- **Why:** Understanding the fixed account ordering is critical for `Seed::AccountKey { index: 3 }`.
- **Fixed order in transfer hook CPI:**

| Index | Account |
|---|---|
| 0 | source token account |
| 1 | mint |
| 2 | destination token account |
| 3 | owner/authority |
| 4 | extra account meta list PDA |
| 5+ | extra accounts (your whitelist PDA) |

### 4.5 `check_is_transferring` Pattern
- **Why:** Prevents someone from calling your `transfer_hook` instruction directly (outside of a real transfer).
- **How:** Checks the `TransferHookAccount.transferring` flag, which Token2022 sets to `true` only during an active transfer.

---

## Phase 5: Read the Project Files (in this order)

Now re-read each file with full understanding:

| Order | File | Concepts to notice |
|---|---|---|
| 1 | `state/whitelist.rs` | `#[account]` macro, minimal struct |
| 2 | `instructions/whitelist_operations.rs` | `init`/`close` constraints, `#[instruction]`, PDA seeds |
| 3 | `instructions/mint_token.rs` | Extension constraints, `InterfaceAccount`, `Interface` |
| 4 | `instructions/init_extra_account_meta.rs` | `Seed::Literal`, `Seed::AccountKey`, TLV encoding |
| 5 | `instructions/transfer_hook.rs` | Token account validation, `check_is_transferring`, PDA-as-whitelist |
| 6 | `lib.rs` | Entry points, `Context<T>`, bumps |
| 7 | `tests/whitelist-transfer-hook.ts` | PDA derivation in TS, `createTransferCheckedWithTransferHookInstruction` |

---

## Quick Reference: Key "Aha" Moments

1. **PDA existence = whitelist check.** If `seeds = [b"whitelist", user]` resolves to a valid account, the user is whitelisted. No data lookup needed. If the PDA doesn't exist, Anchor's deserialization fails and the transaction is rejected.

2. **`close = admin` deletes an account** by zeroing its data, transferring all lamports to `admin`, and setting the discriminator to a closed state.

3. **Anchor constraints replace manual CPI.** The old code did `system_program::transfer` + `realloc` manually. The new code uses `init`/`close` and Anchor handles everything.

4. **`ExtraAccountMeta::new_with_seeds` is dynamic.** Instead of hardcoding a single whitelist PDA address, we tell Token2022 *how to derive* the PDA from the transfer's accounts at runtime.

5. **`createTransferCheckedWithTransferHookInstruction`** reads the ExtraAccountMetaList on-chain and auto-appends the required extra accounts to the transfer instruction. No manual key pushing needed.
