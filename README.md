## ANSIBLE
AVIV Naming Service Indexed Blockchain with Lightweight Events
-----------------------------------------
Ansible is a foundational part of VULCAN which is a foundational part of AVIV.
The purpose is to provide a simple, easy to use KV store in the form of a smart contract, which anyone can use to store and manage small (no larger than 4k) chunks of personal data.

The current version of ANSIBLE is located on Rinkeby at 0x95c12906e7df9931431aa849d1f518d0a8f2ac0a

------------------------------
At its core, ANSIBLE is a user managed KV store for arbitrary data with the user always having full control.  However it was designed for mints to use and therefore mints have the ability to set values on behalf of users.  At the moment there is no method to reject a mint or block it from updating any value on behalf of the user.  However the next version will allow a user to set finer grained permissions over data write and data delete operations.

Here is an explanation of ANSIBLE as it has been deployed to rinkeby.  Assume that we have already imported the contract abi into web3js along with the contract address and therefore each API call below is a method of the contract object.

The two most important functions are set and get.  Set allows the user or a mint to set the current value for any kv pair.  Get retrieves the current value v for any key k.  Get does not retrieve historical values and for this reason, each change to the kv store is logged with an event.

This event is the "ValueSet" event and contains 3 values, the account address which is indexed and key and value which are not indexed.  Therefore replaying the logs allows you to access any historical value v for any address a and also provides a definitive index by which one can determine all values of k for the address.

Note that there are globals, any mint can set kv for address(0), but users cannot.  However this is still namespaced because it is indexed to address(0)

------------------------------
The current API has several events, these are
```
ValueSet
```
ValueSet is the most important event, it allows tracking changes and is used by client software to build indexes.  It emits account, key and value as address, string and string respectively.


```
MintCreated
```
When a mint is registered with the system, this event is emitted.
It emits mint as an address


```
MintRevoked
```
When a mint is revoked (mints can revoke themselves or other mints), the MintRevoked method is emitted.  If this happens, clients using ANSIBLE as a source of truth for consensus need to ignore all future information from the mint unless a new MintCreated event is emitted.
Imagine what would happen if a mint lost control of its keys?  This is here to limit the potential damage.


```
FaucetDry
```
ANSIBLE on ETH and ETC refunds gas whenever it can, in order to lower the barrier to entry, this event lets monitors know that know refunds are presently impossible.  The faucet mechanism will likely go away in the final version because of the potential for abuse and denial of service.
This event includes the address of a person who did not get their gas refunded and these events will continue to emit until someone sends some funds to the contract to allow it to refund gas.


```
FundsRecvd
```
Anyone can send funds in the form of ETH, ETC or any ERC20 Token in order to contribute.  In the event it is a native token such as ETH or ETC then this automatically becomes part of the faucet system, which helps lower the barrier to entry for new users by refunding their gas costs.  In the event the funds are an ERC20 or similar token, then the FundsRecvd event allows managers, mints etc to be notified that money is sitting in a contract somewhere and there is a function that can be used to sweep it out.  At the moment this is finder's keepers.  But in the final version there will be a mint driven voting mechanism.  This will not be going away with the final version.

---------------------------------
## API Functions

To understand the API you must understand the annotations which also correspond to solidity modifiers and are used to control access to functions.

The modifier we use are @payable, @nonpayable, @owneronly, @ownerOrMint, @view

A @payable function is one for which payment is expected, ANSIBLE has only the default payable fallback function in order to maintain compatibility with existing systems and also to ensure the faucet system works as intended.

A @nonpayable is a function for which payment is disallowed.  In general tips are appreciated but all of the functionality of ANSIBLE is free to use.

A @view is a read only function that queries current contract state but changes nothing.  These functions are non-payable because nothing is transmitted, it is merely a query between the client and their blockchain source of truth i.e. blockchain provider whether a local instance or a service like infura.

@ownerOnly are functions that can only be called by the owner of ANSIBLE, the owner being the account that deployed the contract.  These are setup functions and general housekeeping duties that will eventually be turned over to a commitee of mints.

@mintOnly are functions that can only be called by mints, presently there are none, but the task of adding and removing mints will be a mintOnly task in the final version.  It is important to note that the owner is also a mint for the purposes of ANSIBLE.



>> Assume ansible. in front of the function calls
```
@nonpayable @owneronly
changeOwner(address newOwner)
```
The initial owner is the account which deployed the contract.  This account is used to bootstrap the contract by adding an initial set of mints.  Once this is completed, the owner can be changed to address(0) to prevent them mantaining too much control.  Only the current owner can call this function.  Owner is an admin and a super mint, anything ANSIBLE can do can be done by the contract owner.
This function will go away during the final version. 


```
@nonpayable
set(address account, string key, string value)
```
This is used to set a value for a key on behalf of an account.
By default every user can set any k/v pair on their own account and mints can set k/v pairs for any account.  In future versions, users will be given tools to manage which mints are allowed to change pairs and which are barred from doing so.  This function is 99% of what any client should be calling when interacting with the contract.

```
@nonpayable @view
get(address account, string key) returns (string value)
```
Returns the current value v for key k as stored in account a.
Globals are represented as belonging to address(0).
This is a view function meaning it is a read only function that queries contract state and charges no gas.  This makes it the most economical way to sync small bits of data around the world.


```
@mintOnly
setMintState(address account, state bool);
```
This function is used to approve or revoke a mint.  Effectively enabling or disabling any address to function as a mint.  At the moment the owner and/or any mint which is not presently revoked, can approve or revoke a mint.  In future versions this will be a voting mechanism requiring simple majority consent of all mints before the mint is activated or deactivated.


```
@ownerOnly @nonpayable
transferToken(token address, dest address, amount uint)
```
Presently allows only the owner to sweep out any ERC20 tokens that were sent to the contract.
In the future this will be a mint only function and will be voted upon by the active mints.
Votes will be for destination and amount will become a percentage instead of an integer value of smallest divisible units of currency.


```
@view
isMint(account address) returns (bool state)
```
This is a view function to let you know if a given address is currently a mint or not.

----------------------------------------------------------

## Changes in next version

Under the hood, ANSIBLE is written in solidity and features mapping(string=>string) for the personal KV store. These are namespaced by address.
Therefore it is safe to assume the core is composed of... 
```
mapping(address=>mapping(string=>string)) core;
```

Unfortunately, this means that ANSIBLE in its current iteration is not going to work on Ethereum Classic as well as many other chains.

In future versions we will need to update all string types to bytes and arrays of bytes in order to support other chains.  This is because most other chains are on the older solidity v0.4.20 which has flawed support for strings.

Therefore as we move from v 1 to v2 ANSIBLE, for any API call written herein, simply replace string with bytes and derive bytes for K as the keccak hash of the original data being retrieved.

This means that get which is currently
```
ansible.get(address,string) returns (string);
```
Will become...
```
ansible.get(address,bytes32) returns (bytes);
```

And set which is currently
```
ansible.set(address,string,string)
```
Will become
```
ansible.set(address,bytes32,bytes);
```





