# TAP Token Tracking with Trac
This document is supposed to help retrieving indexed & tracked TAP token data for further processing.

Please note that this _confirmed_ data and _not_ mempool data. If you intend to create a marketplace, please make sure to utilize the Bitcoin mempool to prevent frontrunning techniques, especially when it comes to token transfers.

This only holds true for TAP's _external_ functions like "token-deploy", "token-mint" and "token-transfer" as they derive directly from BRC-20 and are a clone of that standard.
_Internal_ functions like "token-send" are mempool-safe and do not require mempool observation.

After public release, you may want to self-host Trac. In this case, the described endpoint below will need to get changed into your own endpoint location.

The examples below use websockets instead of classic RESTful API endpoints as these usually serve large amounts of data better. With the release of Trac, RESTful APIs will be available but should only be used for smaller sets of data.

#### Requirements
- Some Javascript knowledge (Node or Browser)
- Socket.io 4.6.1

#### Setup

HTML/JS

```javascript
<script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.6.1/socket.io.min.js" integrity="sha512-AI5A3zIoeRSEEX9z3Vyir8NqSMC1pY7r5h2cE+9J6FLsoEmSSGLFaqMQw8SWvoONXogkfFrkQiJfLeHLz3+HOg==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
```

Node

```npm install socket.io-client```

Code Anatomy

```javascript
// node only!
const { io } = require("socket.io-client");

// connect to public Bitmap endpoint with Trac

const trac = io("https://tap.trac.network", {
    autoConnect : true,
    reconnection: true,
    reconnectionDelay: 500,
    econnectionDelayMax : 500,
    randomizationFactor : 0
});

trac.connect();

// default response event for all endpoint calls.
// this event handles all incoming results for requests performed using "emit".
//
// You may use the response to render client updates or store in your local database cache.
//
// Example response object: 
// {
//    "error": "",
//    "func": ORIGINALLY-CALLED-FUNCTION-NAME,
//    "args": ARRAY-WITH-ARGUMENTS,
//    "call_id": OPTIONAL-CUSTOM-VALUE,
//    "result": RETURNED-MIXED-TYPE-VALUE
// }

trac.on('response', async function(msg){
  console.log(msg);
});

// default error event for internal Trac errors, if any

trac.on('error', async function(msg){
    console.log(msg);
});

// example getter to get the transaction size of a block
// the results for this call will be triggered by the 'response' event above.
// this structure is exactly the same for all available getters of this endpoint.

trac.emit('get',
{
    func : 'deployment', // the endpoints function to call
    args : ['-tap'],     // the arguments for the function (in this case only 1 argument)
    call_id : ''         // a mixed-type custom id that is passed through in the 'response' event above to identify for which call the response has been and how to proceed.
});
```

#### Composing Getters

You may want to do to things like retrieving list lengths + list items in one operation.
Since the endpoint is using a streaming client and strategy, it is recommended to utilize chained getter calls.

Get account tokens example:

```javascript
async function tokens(address)
{
    trac.emit('get',
    {
        func : 'accountTokensLength',
        args : [address],
        call_id : { cmd : 'tokens_accountTokens', address : address }
    });
}

async function getAccountTokens(address, offset = 0, max = 500, call_id = null)
{
    trac.emit('get',
    {
        func : 'accountTokens',
        args : [address, offset, max],
        call_id : call_id === null ? '' : call_id
    });
}

tokens(address);
```

Processing the response of above calls:

```javascript
trac.on('response', async function(msg){

    if(typeof msg.call_id === 'object')
    {
        switch(msg.call_id.cmd)
        {
            case 'tokens_accountTokens':

                getAccountTokens(
                    msg.call_id.address,
                    0,
                    500,
                    { cmd : 'tokens_end', address : msg.call_id.address }
                );

                break;

            case 'tokens_end':

                console.log(msg);

                // render results...

                break;
        }
    }
)};
```

As you can see, 'tokens_accountTokens' being passed as custom object in the call_id, is sent with the request.
In the response event, the exact call_id will be passed through and you may use the results to trigger the next call in the chain until you received all desired data sets.

#### Available endpoint getters

The results for the calls of the below getters should be used in the "response" event above, explained in Code Anatomy.

Parallel calls are encouraged. Especially if large amounts of data are requested, it is recommended to request them in parallel and smaller chunks.

```javascript
/**
* Expects a ticker string argument.
* Returns a deployment object, containing the deployed token information.
* Returns null if the ticker doesn't exist.
*
* Example returned object:
*
*{
*  tick: 'bitmap',
*  max: '21000000000000000000000000',
*  lim: '1000000000000000000000',
*  dec: 18,
*  blck: 802027,
*  tx: '1e69ce0d3b46a0e2556b01afc87cd596be89c341dfc3e50cd81580a522a77d4a',
*  ins: '1e69ce0d3b46a0e2556b01afc87cd596be89c341dfc3e50cd81580a522a77d4ai0',
*  num: 22023299,
*  ts: 1691376381,
*  addr: 'bc1p6wynl030xsvr48r6kgw99jdtll7jd9n3e8f60j04hmgnrwwysruqwrf9pm',
*  crsd: false
*}
*/ 
trac.emit('get',
{
    func : 'deployment',
    args : [ticker],
    call_id : ''
});

/**
* Returns the amount of deployed tokens.
*/ 
trac.emit('get',
{
    func : 'deploymentsLength',
    args : [],
    call_id : ''
});

/**
* Expects an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 deployment objects, containing the deployed token information.
*/ 
trac.emit('get',
{
    func : 'deployments',
    args : [offset, max],
    call_id : ''
});

/**
* Expects a ticker string argument.
* Returns a string representing the amount of minted tokens in a Big Integer format.
* Returns null if the ticker doesn't exist.
*
* In general, all token balances, transferables, limits and token amounts for this endpoint return in the said format.
*
* You may use JS' BigInt() function to perform calculations.
*
* To render the tokens left in a readable format, you need to retrieve the "dec" (=decimals) attribute from a deployment object (see above).
* Then pass the decimal points to a function that renders a readable format.
*
* Example function:
*
* function formatBigIntString(string, decimals) {
* 
*     let pos = string.length - decimals;
* 
*    if(decimals == 0) {
*       // nothing
*     }else
*     if(pos > 0){
*         string = string.substring(0, pos) + "." + string.substring(pos, string.length);
*     }else{
*         string = '0.' + ( "0".repeat( decimals - string.length ) ) + string;
*     }
* 
*     return string;
* }
*
*/ 
trac.emit('get',
{
    func : 'mintTokensLeft',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the address balance for that ticker as string in Big Integer format.
* Returns null if there are no balances yet for the address/ticker combination.
*
*/
trac.emit('get',
{
    func : 'balance',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the address transferable amount for that ticker as string in Big Integer format.
* Returns null if there are no balances yet for the address/ticker combination.
*
* Note: there is no "available" gettter. Please calculate the available amounts like this:
*
* available = balance - transferable
*
*/
trac.emit('get',
{
    func : 'transferable',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string ticker.
* Returns the amount of holders for the given ticker.
*
* Note: this getter will also include holders for the given ticker, that once owned but no longer hold.
*/ 
trac.emit('get',
{
    func : 'holdersLength',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 holder addresses per call.
*
* Note: this getter will also return holders that had but now own 0 tokens of the given ticker.
*/ 
trac.emit('get',
{
    func : 'holders',
    args : [ticker, offset, max],
    call_id : ''
});


/**
* Expects a string Bitcoin address.
* Returns the amount of tokens owned by the address.
*
* Note: this getter will also include tokens the address no longer owns.
*/ 
trac.emit('get',
{
    func : 'accountTokensLength',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 string tickers.
*
* Note: this getter will also return tickers the address once owned but no longer holds.
*/ 
trac.emit('get',
{
    func : 'accountTokens',
    args : [address, offset, max],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the amount of mints ever performed by the address.
*
* Note: the result includes mints that failed.
*/ 
trac.emit('get',
{
    func : 'accountMintListLength',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 mint objects.
*
* Note: the result includes mints that failed.
*       you can use this list to render an account mint history.
*
* Example mint object:
* 
* {
*     addr: 'bc1pcjy9dud7hmtzyfd3hrulznxfjrcx6m0kmps36dgp8tut7mvddulqdrxzyq',
*     blck: 802098,
*     amt: '0',
*     bal: '0',
*     tx: '8d8ce31acac3ed85eb5df09bc700ddb14359042fb2015fa20beac1baf7384227',
*     ins: '8d8ce31acac3ed85eb5df09bc700ddb14359042fb2015fa20beac1baf7384227i0',
*     num: -77585,
*     ts: 1691413973,
*     fail: true
* }
*/ 
trac.emit('get',
{
    func : 'accountMintList',
    args : [address, ticker, offset, max],
    call_id : ''
});

/**
* Expects a string ticker.
* Returns the amount of mints ever performed for the given ticker.
*
* Note: the result includes mints that failed.
*/ 
trac.emit('get',
{
    func : 'tickerMintListLength',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 mint objects.
*
* Note: the result includes mints that failed.
*       you can use this list to render a ticker mint history.
*/ 
trac.emit('get',
{
    func : 'tickerMintList',
    args : [ticker, offset, max],
    call_id : ''
});

/**
* Returns the amount of mints ever performed.
*
* Note: the result includes mints that failed.
*/ 
trac.emit('get',
{
    func : 'mintListLength',
    args : [],
    call_id : ''
});

/**
* Expects an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 mint objects.
*
* Note: the result includes mints that failed.
*       you can use this to stream the entire mint history, across all tickers and addresses.
*/ 
trac.emit('get',
{
    func : 'mintList',
    args : [offset, max],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the amount transfer-inscribes ever performed by the address.
*
* Note: the result includes transfers that failed.
*/ 
trac.emit('get',
{
    func : 'accountTransferListLength',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 transfer-inscribe objects.
*
* Note: the result includes transfers that failed.
*       you can use this list to render an account transfer-inscribe history.
*
*/ 
trac.emit('get',
{
    func : 'accountTransferList',
    args : [address, ticker, offset, max],
    call_id : ''
});

/**
* Expects a string ticker.
* Returns the amount of inscribe-transfer inscriptions ever performed for the given ticker.
*
* Note: the result includes mints that failed.
*/ 
trac.emit('get',
{
    func : 'tickerTransferListLength',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 inscribe-transfer objects.
*
* Note: the result includes transfer-inscribes that failed.
*       you can use this list to render a ticker transfer-inscribe history.
*/ 
trac.emit('get',
{
    func : 'tickerTransferList',
    args : [ticker, offset, max],
    call_id : ''
});

/**
* Returns the amount of mints ever performed.
*
* Note: the result includes transfer-inscribes that failed.
*/ 
trac.emit('get',
{
    func : 'transferListLength',
    args : [],
    call_id : ''
});

/**
* Expects an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 transfer-inscribe objects.
*
* Note: the result includes transfer-inscribes that failed.
*       you can use this to stream the entire transfer-inscribe history, across all tickers and addresses.
*/ 
trac.emit('get',
{
    func : 'transferList',
    args : [offset, max],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the amount sends ever performed by the address.
*
* Note: the result includes sends that failed.
*/ 
trac.emit('get',
{
    func : 'accountSentListLength',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 send objects.
*
* Note: the result includes sends that failed.
*       you can use this list to render an account send history.
*
*/ 
trac.emit('get',
{
    func : 'accountSentList',
    args : [address, ticker, offset, max],
    call_id : ''
});

/**
* Expects a string ticker.
* Returns the amount of sends ever performed for the given ticker.
*
* Note: the result includes sends that failed.
*/ 
trac.emit('get',
{
    func : 'tickerSentListLength',
    args : [ticker],
    call_id : ''
});

/**
* Expects a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 send objects.
*
* Note: the result includes sends that failed.
*       you can use this list to render a ticker send history.
*/ 
trac.emit('get',
{
    func : 'tickerSentList',
    args : [ticker, offset, max],
    call_id : ''
});

/**
* Returns the amount of sends ever performed.
*
* Note: the result includes sends that failed.
*/ 
trac.emit('get',
{
    func : 'sentListLength',
    args : [],
    call_id : ''
});

/**
* Expects an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 send objects.
*
* Note: the result includes sends that failed.
*       you can use this to stream the entire send history, across all tickers and addresses.
*/ 
trac.emit('get',
{
    func : 'sentList',
    args : [offset, max],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the amount of tokens ever received by the address.
*
* Note: the result includes receives that failed.
*/ 
trac.emit('get',
{
    func : 'accountReceiveListLength',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 receive objects.
*
* Note: the result includes receives that failed.
*       you can use this list to render an account receive history.
*
*/ 
trac.emit('get',
{
    func : 'accountReceiveList',
    args : [address, ticker, offset, max],
    call_id : ''
});

/**
* Expects an inscription id (not number)
* Returns an accumulation object for internal functions, including the inscribed json object.
* Returns null if the accumulator object doesn't exist.
*/
trac.emit('get',
{
    func : 'accumulator',
    args : [inscription id],
    call_id : ''
});

/**
* Expects a string Bitcoin address and a string ticker.
* Returns the amount of accumulated items, which represent incoming internal function prior tapping.
*
* Note: the result includes accumulated items that may have been tapped already.
*/ 
trac.emit('get',
{
    func : 'accountAccumulatorListLength',
    args : [address, ticker],
    call_id : ''
});

/**
* Expects a string Bitcoin address, a string ticker, an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 accumulator objects.
*
* Note: the result includes accumulated items that may have been tapped already.
*       to check if the an accumulated item has been tapped, check for its presence using the getter "accumulator" above.
*       if a returned accumulator from the list is null, it means it has been tapped.
*       you can use this list to render an account accumulation history and present tappable internal functions (=awaiting confirmation).
*
*/ 
trac.emit('get',
{
    func : 'accountAccumulatorList',
    args : [address, ticker, offset, max],
    call_id : ''
});

/**
* Returns the amount of accumulated items across all addresses, which represent incoming internal function prior tapping.
*
* Note: the result includes accumulated items that may have been tapped already.
*/ 
trac.emit('get',
{
    func : 'accumulatorListLength',
    args : [],
    call_id : ''
});

/**
* Expects an integer offset (min 0) to start from and an integer max (max 500).
* Returns up to 500 send objects.
*
* * Note: the result includes accumulated items that may have been tapped already.
*       to check if the an accumulated item has been tapped, check for its presence using the getter "accumulator" above.
*       if a returned accumulator from the list is null, it means it has been tapped.
*       you can use this list to render a "mempool-like" accumulation history and present tappable internal functions (=awaiting confirmation).
*/ 
trac.emit('get',
{
    func : 'accumulatorList',
    args : [offset, max],
    call_id : ''
});
```
