# Readme
This document outlines a portion of the [Vellum](https://onvellum.com) program as it pertains to the post execution feature. You will find an accompanying [Anchor IDL](vellum.json) to aid your development.

## Program Info
### Program ID
`zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX`
### Instructions
#### `createPostExecutionRoute`
```javascript
/* SOME IMPORTS, VARIABLES, LINES OMITTED FOR BREVITY */
import VellumIDL from './vellum.json';

const program = new anchor.Program(
  VellumIDL,
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
  { connection: new Connection('https://api.mainnet-beta.solana.com') },
);

const instruction = await program.methods
  .createPostExecutionRoute(
    agreement.id,
    routeIndex,
    [],
    routeType,
  )
  .accounts({
    payer,
    agreement: agreementPda,
    postExecutionRoute: routePda,
    systemProgram: SystemProgram.programId,
  })
  .instruction();
```
#### `createPostExecutionRouteEscrow`
```javascript
/* SOME IMPORTS, VARIABLES, LINES OMITTED FOR BREVITY */
import VellumIDL from './vellum.json';

const program = new anchor.Program(
  VellumIDL,
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
  { connection: new Connection('https://api.mainnet-beta.solana.com') },
);

const instruction = await program.methods.createPostExecutionRouteEscrow(
  routePdaBump,
  routeIndex,
)
  .accounts({
    payer: wallet,
    agreement: agreementPda,
    postExecutionRoute: routePda,
    postExecutionRouteEscrow: routeEscrowPda,
    escrowMint: mint,
    tokenProgram: TOKEN_PROGRAM_ID,
    rent: SYSVAR_RENT_PUBKEY,
    systemProgram: SystemProgram.programId,
  })
  .instruction();
```
#### `depositPostExecutionRouteEscrow`
```javascript
/* SOME IMPORTS, VARIABLES, LINES OMITTED FOR BREVITY */
import VellumIDL from './vellum.json';

const program = new anchor.Program(
  VellumIDL,
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
  { connection: new Connection('https://api.mainnet-beta.solana.com') },
);

const [delegator, escrowMint] = await (async () => {
  switch (Object.keys(routePdaAccount.routeType)[0]) {
    case "nativeMint": {
      return [
        wallet,
        NATIVE_MINT,
      ];
    }
    case "tokenEscrow": {
      return [
        await getAssociatedTokenAddress(
          routePdaAccount.routeType["tokenEscrow"].mint,
          wallet,
        ),
        routePdaAccount.routeType["tokenEscrow"].mint,
      ];
    }
  }
})();

const instruction = await program.methods.depositPostExecutionRouteEscrow(
  routePdaBump,
  routeEscrowPdaBump,
  routeIndex,
)
  .accounts({
    owner: wallet,
    delegator,
    agreement: agreementPda,
    postExecutionRoute: routePda,
    postExecutionRouteEscrow: routeEscrowPda,
    escrowMint,
    tokenProgram: TOKEN_PROGRAM_ID,
    systemProgram: SystemProgram.programId,
  })
  .instruction();
```
#### `processPostExecutionRouteEscrow`
```javascript
/* SOME IMPORTS, VARIABLES, LINES OMITTED FOR BREVITY */
import VellumIDL from './vellum.json';

const program = new anchor.Program(
  VellumIDL,
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
  { connection: new Connection('https://api.mainnet-beta.solana.com') },
);

const [routePda, routePdaBump] = await PublicKey.findProgramAddress(
  [
    agreement.id.toBuffer(),
    Buffer.from('routes'),
    new anchor.BN(routeIndex).to('le'),
  ],
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
);

const routePdaAccount =
  await program.account.postExecutionRoute.fetch(routePda);

// construct flat accounts meta from nested instructions
const routePdaAccountInstructionsAccountsMeta = 
  routePdaAccount.instructions.map(({ accounts }) => {
    return accounts.map(({ pubkey, isWritable }) => ({
      pubkey,
      isWritable,
      false,
    })).flat();
  }).flat();

const instruction = await program.methods.processPostExecutionRouteEscrow(
  routePdaBump,
  routeEscowPdaBump,
  routeIndex,
)
  .accounts({
    payer: wallet,
    agreement: agreementPda,
    postExecutionRoute: routePda,
    postExecutionRouteEscrow: routeEscowPda,
    escrowMint: NATIVE_MINT,
    systemProgram: SystemProgram.programId,
    tokenProgram: TOKEN_PROGRAM_ID,
  })
  .remainingAccounts(
    [
      ...routePdaAccountInstructionsAccountsMeta,
      Object.keys(routePdaAccount.routeType)[0] === "tokenEscrow" ? {
        pubkey: TOKEN_PROGRAM_ID,
        isWritable: true,
        isSigner: false,
      } : undefined,
      Object.keys(routePdaAccount.routeType)[0] === "nativeEscrow" ? {
        pubkey: routePdaAccount.routeType["nativeEscrow"].dst_wallet,
        isWritable: true,
        isSigner: false,
      } : undefined,
    ].filter((account) => account != undefined)
  )
  .instruction();
```
### Accounts
#### `Agreement`
##### State
```rust
pub struct Agreement {
    pub version: u16,
    pub creator: Pubkey,
    pub initializer: Pubkey,
    pub id: Pubkey,
    pub file_name: String,
    pub document_url: String,
    pub remaining_signatures: u32,
    pub signatories: Vec<VellumSigner>,
    pub created_at: i64,
    pub bump: u8,
    pub quorum: u32,
    pub public: bool,
    pub post_execution: Option<PostExecution>,
}
```
#### `PostExecutionRoute`
##### State
```rust
pub struct PostExecutionRoute {
    pub index: u8,
    pub route_type: RouteType,
    pub agreement: Pubkey,
    pub instructions: Vec<InstructionData>,
    pub executed: bool,
    pub received: bool,
}
```
##### Seeds
```javascript
const [routePda, routePdaBump] = await PublicKey.findProgramAddress(
  [
    agreement.id.toBuffer(),
    Buffer.from('routes'),
    new anchor.BN(routeIndex).to('le'),
  ],
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
);
```
#### `PostExecutionRouteEscrow`
##### State ([SPL Token Account](https://github.com/solana-labs/solana-program-library/blob/f825f1d10672275fad00dc8c1ba4658c9b53f2c1/token/program/src/state.rs#L86))
```rust
pub struct Account {
    /// The mint associated with this account
    pub mint: Pubkey,
    /// The owner of this account.
    pub owner: Pubkey,
    /// The amount of tokens this account holds.
    pub amount: u64,
    /// If `delegate` is `Some` then `delegated_amount` represents
    /// the amount authorized by the delegate
    pub delegate: COption<Pubkey>,
    /// The account's state
    pub state: AccountState,
    /// If is_native.is_some, this is a native token, and the value logs the rent-exempt reserve. An
    /// Account is required to be rent-exempt, so the value is used by the Processor to ensure that
    /// wrapped SOL accounts do not drop below this threshold.
    pub is_native: COption<u64>,
    /// The amount delegated
    pub delegated_amount: u64,
    /// Optional authority to close the account.
    pub close_authority: COption<Pubkey>,
}
```
##### Seeds
```javascript
const [routeEscrowPda, routeEscrowPdaBump] = await PublicKey.findProgramAddress(
  [
    agreement.id.toBuffer(),
    Buffer.from('routes'),
    new anchor.BN(routeIndex).to('le'),
    mint.toBuffer(),
  ],
  new PublicKey('zdadjXVxQcdn95i1dQ9T6F8dzN8cAXJuHzhrjs22VMX'),
);
```
### Types
#### `PostExecution`
```rust
pub struct PostExecution {
    pub routes: Vec<Pubkey>,
    pub executions: Vec<bool>,
}
```
#### `RouteType`
```rust
pub enum RouteType {
    NativeEscrow {
        // (w)SOL escrows
        delegator: Pubkey, // wallet responsible for funding
        dst_wallet: Pubkey,
        amount: u64,
    },
    TokenEscrow {
        // spl-token escrows
        mint: Pubkey,
        delegator: Pubkey, // token ata responsible for funding
        dst_wallet: Pubkey,
        amount: u64,
    },
    Custom {
        // 140 bytes
        short_description: String,
    },
}
```
#### `VellumSigner`
```rust
pub struct VellumSigner {
    pub key: Pubkey,
    pub name: String,
    pub organization: String,
    pub email: String,
    pub signature: String,
    pub signed_at: i64,
    pub ip: String,
    pub tx_signature: String,
}
```
