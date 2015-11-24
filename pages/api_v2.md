## Overview
Welcome to **Cryptotrader.org API**. We aim to provide an API that allows developers to write 
fully featured trading algorithms. Our automated trading platform can backtest and execute trading scripts coded in CoffeeScript on historical data.


## Trading API

Each script has to implement the following two methods:

#### init
Initialization method called before trading logic starts. Put any initizalition here.
    
**Example:**

    init: ->
      @context.treshold = 0.21  
      ...  

#### handle
Called on each tick according to the tick interval that was set (e.g 1 hour)


**Example:**

    handle: ->
      ...
      # pick default instrument
      instrument = @data.instruments[0]
      debug "Price: #{instrument.price}"




### Global data and methods

#### @context
object that holds current script state

#### @data
provides access to current trading environment
- @data.at - current time in milliseconds
      
- @data.instruments - array of instrument objects (see Instrument object below). *Currenly only single instrument object is supported*
 

#### @storage 
the object is used to persist variable data in the database that is essential for developing reliable algorithms. In the case if an instance is interrupted unexpectedly, the storage object will be restored from the database to the last state. Do not use to store complex objects such as classes and large amounts for data (> 64kb).

  **Example:**

          handle: ->
            ...
            if orderId
              @storage.lastBuyPrice = instrument.price

#### stop
  The method immediately stops script execution and updates the log with the resulting balance.
  
#### onRestart
  This hook method is called upon restart

**Example**

  onRestart: ->
   
      debug "Restart detected"

#### onStop
This hook method is called when the bot stops


**Example**

    onStop: ->  
    	debug "The bot will be stopped"

### Instrument object
The object that provides access to current trading data and technical indicators.

- OHLCV candlestick data. Includes the following array data

  - open
  - low
  - high
  - close
  - volumes
  
**Example:**

    instrument = @data.instrument[0]
    debug instrument.high[instrument.high.length-1] # Displays current High value

#### market 
  the market/exchange this instrument is traded on

#### period
  trading period in minutes 

#### price 
  current price

#### volume
  current volume

#### ticker
  provides access to live ticker data and has two properties: ticker.buy and ticker.sell
  gets updated each time before handle method is called or after an order is traded




### Core Modules 

#### Logger

#### debug(msg), info(msg), warn(msg)
Logs message with specified log level 

#### Plot


#### plot(series)
  Records data to be shown on the chart as line graphs. Series is a dictionary object that includes key and value pairs.

**Example**

    # draw moving averages on the chart
    plot
        short: instrument.ema(10)
        long: instrument.ema(21)

#### plotMark(marks)
  Similar to plot method, but adds a marker that displays events on the charts.

**Example**

    high = _.last instrument.high
    if high > maxPrice
      plotMark
          "new high": high


#### setPlotOptions(options)
  This method allows to configure how series data and markers will be plotted on the chart. Currently the following options are supported:
  
  - color (in HTML format. e.g #eeeeee or 'darkblue')
  - lineWidth (default: 1.5)
  - secondary (default: false) specifies whether the series should plotted to secondary y-axis

**Example**
  
    init: ->
      ...
      setPlotOptions
        sell:
          color: 'orange'
        EMA:
          color: 'deeppink'
          lineWidth: 5


#### _ [lodash library]
  The binding to Lodash library (http://lodash.com/) which has many helper functions
  
**Example**

    debug _.max(instrument.high.splice(-24)) # Prints maximum high for last 24 periods

### Modules 

Non-core modules shoud be imported using **require** directive before they can be used.

#### Ta-lib
  The interface to Ta-lib library (http://ta-lib.org/), which contains 125+ indicators for technical analysis.
  [TA-Lib Functions (Index)](/talib)
  

**Example**

    talib = require 'talib'
    # Calculate SMA(10) using ta-lib API
    instrument = @data.instruments[0]
    value = talib.SMA
      startIdx: 0 
      endIdx: instrument.close.length-1
      inReal: instrument.close
      optInTimePeriod: 10
      

#### Params 
The module allows to create bots that take user input at runtime.

##### add title, defaultValue

In order to pass parameters to your strategy, put **add** method call to the top of the code, like in the example below:


	params = require 'params
    STOP_LOSS = params.add 'Stop Loss',100 # default value is 100
    MARKET_ORDER = params.add 'Market Order',false # will be displayed as checkbox
    MODE = params.add 'a) Low risk b) Aggressive','a' # can be a string value


#### Bitfinex Margin Trading (bitfinex/margin_trading)

This module enables leveraged trading on Bitfinex

##### getMarginInfo(instrument)
Returns your trading wallet information for margin trading:
- margin_balance - the USD value of all your trading assets
- tradable_balance - Your tradable balance in USD (the maximum size you can open on leverage for this pair)

**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = @data.instruments[0]
      info = mt.getMarginInfo instrument
      debug "price: "margin balance: #{info.margin_balance} tradeable balance: #{info.tradable_balance}"

##### getPosition(instrument)
Returns the active position for specified instrument 

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = @data.instruments[0]
	  pos = mt.getPosition instrument
      if pos
        debug "position: #{pos.amount} @#{pos.price}"

##### closePosition(instrument)
Closes open position

##### buy(instrument,[amount],[price],[timeout])
This method executes a purchase of specified asset. 

*timeout* parameter allows to limit the length of time in seconds an order can be outstanding before being canceled. 


**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open long position 
      if mt.buy instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'BUY order traded'

#### sell(instrument,[amount],[price],[timeout])
This method executes a sale of specified asset. 

**Example:**

    mt = require "bitfinex/margin_trading"
    
    handle: ->
      instrument = data.instruments[0]
      info = mt.getMarginInfo instrument
      ## Open short position 
      if mt.sell instrument,info.tradable_balance/instrument.price,instrument.price
        debug 'SELL order traded'


**Advanced orders **

This set of functions gives more control over how orders are being processed:

#### addOrder(order)

Submits a new order and returens an object containing information about order

The order parameter is an object contaning:
- instrument - current instrument
- side - order side "buy" or "sell"
- type - 'stop' or 'limit'
- amount - order amount
- price - order price

**Returns:**

- id - unique orderId. Note that id can be missing if the order was filled instantly.
- active - true if the order is still active
- cancelled - true if the order was cancelled
- filled - true if the order was traded

The engine automatically tracks all active orders and peridically update their statuses.

**Example**

    	...
		stopOrder = mt.addOrder 
  		instrument: instrument
  		side: 'buy'
  		type: 'stop'
  		amount: amount
  		price: instrument.price * 1.05
        if order.id
        	debug "Order Id: #{stopOrder.id}"
        else
        	debug "Order fulfilled"

#### getActiveOrders
Returns the list of currently open orders

#### getOrder(orderId)

Returns an order object by given id.

#### cancelOrder(order)

Cancels an order.

#### linkOrder(orderA,orderB)
Links orders so when one of them is cancelled or closed, the other one will be cancelled automatically.

   



