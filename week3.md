## Security analysis - Solana program
1. transfer result ignored
1. from == to

## Security analysis - Soroban contract
### H-01 Burn Missing Access Control

### H-02 transfer_from does not clear approval

### H-03 admin is stored as temporary variable

### M-01 ID-based approvals cannot be revoked

### L-01 `get_all_owned` view function will run out of gas

### L-02 Ambiguous event emission for approve_all

### L-03 Instance storage TTL extended but not used

## Security analysis - Node implementation

> Research and explain what do lines 57 to 60 do?

Registers a hook function to be called when a thread panics. When one of the handlers throws an exception, this hook function gets called. The problem is that the hook function causes the process to abort on line 59: any unhandeled exception will cause the server to crash because of this.

### H-01 Divide by zero error crashes node

### H-02 Unbounded memory usage crashes node