# Autonomy Network Frontend Defi Dapp Engineer Assessment

Thank you for taking the time to apply to the Autonomy Network. As we are a smaller startup, the founding team is looking for someone that can work with little direction and supervision. We want someone that can come up with original ideas and execute them. This will help us push the Autonomy Network as the go-to solution for all decentralized conditional automation.

First, I would advise that you take a look at our website and medium:

https://www.autonomynetwork.io
https://autonomynetwork.medium.com 

Here is our contract addresses that have already been deployed to the Ropsten Network:

Registry Address:
0xB82Ae7779aB1742734fCE32A4b7fDBCf020F2667

If you need any further information or need some smart contracts done for your project please contact 
@quantafire on the Autonomy discord.

If you have any other questions feel free to message us on our discord channel:

https://discord.gg/XFEsHGhsxc

Once you feel more familiar with what we are building at the Autonomy Network, you can start brainstorming ideas (eg. NFT automation, recurring subscription payments, limit orders on a DEX like Kyber or Bancor (not Uniswap since that example is below ;) ), etc). You can even bounce ideas off of us, again just reach us on our discord. (Clear communication is an asset 👀👀)

After you come up with an idea, scope it out to your preference and current time commitments. Submit the scope or just let us know what is the timeline of your project. After that you can start coding.

It is very important to mention that we are not looking for fully fledged projects, it can be a proof of concept or a mock project, we just want to know that you have the technical expertise and the autonomy to work with little to no supervision and make it look good at the same time. (If we like the idea and it works we will compensate you for your time, so make sure you scope it out properly).

The project must:
 - use Autonomy to automate some user action under some condition in the future
 - use MetaMask to interact with the user's account
 - have a nice looking UI

Finally, you can submit your project by sending us the code (git repo preferred, but you can send us files too) to @caraluno on Discord. Extra points for well documented code. You can deploy it or just give us instructions to look at the project in a dev environment, a small description of what you built would be amazing. Again you can reach us on discord or email for submissions.


Good luck, we are looking forward to your submission,

Autonomy Network Team

# How to use Autonomy
The architecutre of Autonomy that you need to be aware of is that there's a Registry contract where users make a transaction to in order to register a new request. These requests have a condition built into the contracts they call which makes the transaction revert if the condition is false. The decentralised network of executor bots monitor all the requests in the Registry contract for when the requests can be executed without reverting.


A simple example is if you wanted to create a dapp that sends eth to someone at a specific time in the future.
In order to add a condition that reverts, you would need to create this simple contract:
```
contract EthSender {
  function sendEthAtTime(uint time, address payable recipient) external payable {
    require(block.timestamp >= time, "Too soon");
    recipient.transfer(msg.value);
  }
}
```
A deployed instance of the above contract is here https://ropsten.etherscan.io/address/0xfa0a8b60b2af537dec9832f72fd233e93e4c8463

You would need to make and send a transaction to `newReq` in the Registry (0xB82Ae7779aB1742734fCE32A4b7fDBCf020F2667), which is the only function that you need to care about in the frontend that you actually send a transaction to. The interface is:
```
    /**
     * @notice  Creates a new request, logs the request info in an event, then saves
     *          a hash of it on-chain in `_hashedReqs`
     * @param target    The contract address that needs to be called
     * @param referer       The referer to get rewarded for referring the sender
     *                      to using Autonomy. Usally the address of a dapp owner
     * @param callData  The calldata of the call that the request is to make, i.e.
     *                  the fcn identifier + inputs, encoded
     * @param ethForCall    The ETH to send with the call
     * @param verifySender  Set to false
     * @param payWithAUTO   Whether the sender wants to pay for the request in AUTO
     *                      or ETH. Paying in AUTO reduces the fee
     * @return id   The id of the request, equal to the index in `_hashedReqs`
     */
    function newReq(
        address target,
        address payable referer,
        bytes calldata callData,
        uint120 ethForCall,
        bool verifySender,
        bool payWithAUTO
    ) external payable returns (uint id);
```

The logic flow would be this:
1. The frontend takes in the user inputs, and generates the calldata needed to call this contract, e.g.:
  `callData = ethSender.methods.sendEthAtTime(time, userAddress).encodeABI()`
  Note that no transaction has been sent to the blockchain at this point! The information needed to call this function in the future was generated
2. Call `newReq` at Registry with the following parameters:
 - `target` would be equal to the EthSender contract address
 - `referer` can just be set to `0x00..00`
 - `callData` would be the `callData` generated in step 1.
 - `ethForCall` would be the amount of eth the user wants to send in the future
 - `verifySender` would be `false` since EthSender doesn't need to know who the sender is, it only cares who the recipient is
 - `payWithAUTO` would be `false`
 - 'value' would be `ethForCall` + 0.01 ETH. The 0.01 ETH is because more ETH needs to be sent to pay for the bot to execute the transaction. On Ropsten, 0.01 ETH above `ethForCall` should be more than enough - any excess that isn't used to pay the executing bot will get sent back to the user.

Send this transaction to the blockchain - this is the 1st and only transaction that the user sends
In this case, `ethForCall` is the amount of ETH to send to `sendEthAtTime` and therefore is how much ETH we want to send to the recipient - so this is the amount inputted by the user in the UI.
3. When the condition is met, a bot executes the request. The bot pays the gas fees with the ETH sent with the request in 2. and any excess ETH is returned to the user. The `recipient` receives the ETH in this transaction. This is the 2nd transaction sent to the blockchain, but is not sent by the user directly.


A slightly more complicated example is, if Uniswap wanted to add limit orders to their DEX, they could create a wrapper contract that adds the condition 'if price is below x, revert'. The user would just click a limit order button, and the frontend would make a transaction to the Registry, calling `newReq`.
An example of this can be seen in https://github.com/Autonomy-Network/AutoSwap-Diff/pull/1
It is live here: https://autoswap-mock-deployment.vercel.app/#/swap

Generally to keep things as simple as possible, the parameters would be:
 - `target` would be equal to the Uniswap wrapper contract
 - `referer` can just be set to `0x00..00`
 - `callData` would be the calldata that would need to be sent to the wrapper contract. This can be seen with https://github.com/Autonomy-Network/AutoSwap-Diff/blob/923fc0feb300f429b93221c9fcbae97ef889beae/src/hooks/useSwapCallback.ts#L237
 - `ethForCall` would be 0 if no eth is being sent. If the user wanted to trade eth > token, then `ethForCall` should be the amount of eth they want to trade, in wei
 - `verifySender` would be `true` since you want the wrapper contract to know who the user is
 - `payWithAUTO` would be `false`
The 'value' sent with the function call would have to be equal to `ethForCall` to call the target function, and also have enough eth to pay the executor bot for the eth they spent executing the request in the future, plus a small fee (30% of the total gas cost that the executor paid). So regardless of whether `ethForCall` is 0, 'value' = `ethForCall` + 0.01 ETH (depends on gas prices, but for Ropsten, 0.01 ETH is enough to cover this. Any excess ETH that is not used in the execution is returned to the user after execution has finished)
