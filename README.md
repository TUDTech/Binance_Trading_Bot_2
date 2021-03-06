# Binance Trading Bot

Automated Binance trading bot with trailing buy/sell strategy

---

## Trailing Buy/Sell Bot

This bot is using the concept of trailing buy/sell order which allows following
the price fall/rise.

- The bot can monitor multiple symbols. Each symbol will be monitored per
  second.
- The bot is only tested and working with USDT pair in the FIAT market such as
  BTCUSDT, ETHUSDT. You can add more FIAT symbols like BUSD, AUD from the
  frontend. However, I didn't test in the live server. So use with your own
  risk.
- The bot is using MongoDB to provide a persistence database. However, it does
  not use the latest MongoDB to support Raspberry Pi 32bit.
  
## Buy Signal

The bot will continuously monitor the lowest value for the period of the
candles. Once the current price reaches the lowest price, then the bot will
place a STOP-LOSS-LIMIT order to buy. If the current price continuously falls,
then the bot will cancel the previous order and re-place the new STOP-LOSS-LIMIT
order with the new price.

- The bot will not place a buy order if has enough coin (typically over $10
  worth) to sell when reaches the trigger price for selling.

## Buy Scenario

Let say, if the buy configurations are set as below:

- Maximum purchase amount: $50
- Trigger percentage: 1.005 (0.5%)
- Stop price percentage: 1.01 (1.0%)
- Limit price percentage: 1.011 (1.1%)

And the market is as below:

- Current price: $101
- Lowest price: $100
- Trigger price: $100.5

Then the bot will not place an order because the trigger price ($100.5) is less
than the current price ($101).

In the next tick, the market changes as below:

- Current price: $100
- Lowest price: $100
- Trigger price: $100.5

The bot will place new STOP-LOSS-LIMIT order for buying because the current
price ($100) is less than the trigger price ($100.5). The new buy order will be placed as below:

- Stop price: $100 \* 1.01 = $101
- Limit price: $100 \* 1.011 = $101.1
- Quantity: 0.49

In the next tick, the market changes as below:

- Current price: $99
- Current limit price: $99 \* 1.011 = 100.089
- Open order stop price: $101

As the open order's stop price ($101) is higher than the current limit price
($100.089), the bot will cancel the open order and place new STOP-LOSS-LIMIT
order as below:

- Stop price: $99 \* 1.01 = $99.99
- Limit price: $99 \* 1.011 = $100.089
- Quantity: 0.49

If the price continuously falls, then the new buy order will be placed with the
new price.

And if the market changes as below in the next tick:

- Current price: $100

Then the current price reaches the stop price ($99.99); hence, the order will be
executed with the limit price ($100.089).

## Sell Signal

If there is enough balance for selling and the last buy price is recorded in the
bot, then the bot will start monitoring the sell signal. Once the current price
reaches the trigger price, then the bot will place a STOP-LOSS-LIMIT order to
sell. If the current price continuously rises, then the bot will cancel the
previous order and re-place the new STOP-LOSS-LIMIT order with the new price.

- If the coin is worth less than typically $10 (minimum notional value), then
  the bot will remove the last buy price because Binance does not allow to place
  an order of less than $10.
- If the bot does not have a record for the last buy price, the bot will not
  sell the coin.

## Sell Scenario

Let say, if the sell configurations are set as below:

- Trigger percentage: 1.05 (5.0%)
- Stop price percentage: 0.98 (-2.0%)
- Limit price percentage: 0.979 (-2.1%)

And the market is as below:

- Coin owned: 0.5
- Current price: $100
- Last buy price: $100
- Trigger price: $100 \* 1.05 = $105

Then the bot will not place an order because the trigger price ($105) is higher
than the current price ($100).

If the price is continuously falling, then the bot will keep monitoring until
the price reaches the trigger price.

In the next tick, the market changes as below:

- Current price: $105
- Trigger price: $105

The bot will place new STOP-LOSS-LIMIT order for selling because the current
price ($105) is higher or equal than the trigger price ($105). For the simple
calculation, I do not take an account for the commission. In real trading, the
quantity may be different. The new sell order will be placed as below:

- Stop price: $105 \* 0.98 = $102.9
- Limit price: $105 \* 0.979 = $102.795
- Quantity: 0.5

In the next tick, the market changes as below:

- Current price: $106
- Current limit price: $103.774
- Open order stop price: $102.29

As the open order's stop price ($102.29) is less than the current limit price
($103.774), the bot will cancel the open order and place new STOP-LOSS-LIMIT
order as below:

- Stop price: $106 \* 0.98 = $103.88
- Limit price: $106 \* 0.979 = $103.774
- Quantity: 0.5

If the price continuously rises, then the new sell order will be placed with the
new price.

And if the market changes as below in the next tick:

- Current price: $103

The the current price reaches the stop price ($103.88); hence, the order will be
executed with the limit price ($103.774).

## Frontend + WebSocket

React.js based frontend communicating via Web Socket:

- List monitoring coins with buy/sell signals/open orders
- View account balances
- Manage global/symbol settings
- Delete caches that are not monitored
- Link to public URL
- Support Add to Home Screen

## Environment Parameters

Use environment parameters to adjust parameters. Check
`/config/custom-environment-variables.json` to see list of available environment
parameters.

## How to use

1. Create `.env` file based on `.env.dist`.

   | Environment Key                | Description                       | Sample Value   |
   | ------------------------------ | --------------------------------- | -------------- |
   | BINANCE_LIVE_API_KEY           | Binance API key for live          | (from Binance) |
   | BINANCE_LIVE_SECRET_KEY        | Binance API secret for live       | (from Binance) |
   | BINANCE_TEST_API_KEY           | Binance API key for test          | (from Binance) |
   | BINANCE_TEST_SECRET_KEY        | Binance API secret for test       | (from Binance) |
   | BINANCE_SLACK_WEBHOOK_URL      | Slack webhook URL                 | (from Slack)   |
   | BINANCE_SLACK_CHANNEL          | Slack channel                     | "#binance"     |
   | BINANCE_SLACK_USERNAME         | Slack username                    | Jack           |
   | BINANCE_LOCAL_TUNNEL_SUBDOMAIN | Local tunnel public URL subdomain | binance        |

2. Check `docker-compose.yml` for `BINANCE_MODE` environment parameter

3. Launch the application with docker-compose

   ```bash
   git pull
   docker-compose up -d
   ```

   or using the latest build image from DockerHub

   ```bash
   git pull
   docker-compose -f docker-compose.server.yml pull
   docker-compose -f docker-compose.server.yml up -d
   ```

   or if using Raspberry Pi 32bit. Must build again for Raspberry Pi.

   ```bash
   git pull
   docker build . --build-arg NODE_ENV=production --target production-stage -t chrisleekr/binance-trading-bot:latest
   docker-compose -f docker-compose.rpi.yml up -d
   ```

4. Open browser `http://0.0.0.0:8080` to see the frontend

## Screenshots

Frontend Mobile                                                                                                         

![Screen Shot 2021-03-23 at 21 00 20](https://user-images.githubusercontent.com/81108192/112217743-c958a400-8c1a-11eb-9f92-cb8bd4a1b4ce.png)

Setting

![Screen Shot 2021-03-23 at 21 00 49](https://user-images.githubusercontent.com/81108192/112217799-daa1b080-8c1a-11eb-9b37-1b30fc69fb81.png)

Frontend Desktop

![Screen Shot 2021-03-23 at 21 03 29](https://user-images.githubusercontent.com/81108192/112218085-39ffc080-8c1b-11eb-946d-ff0803c20dd0.png)

## Sample Trade

Chart         

![Screen Shot 2021-03-23 at 21 03 59](https://user-images.githubusercontent.com/81108192/112218139-4c79fa00-8c1b-11eb-94ac-e8782ce45213.png)

Buy Orders

![Screen Shot 2021-03-23 at 21 04 37](https://user-images.githubusercontent.com/81108192/112218213-6287ba80-8c1b-11eb-960d-2fb013f1a595.png)

Sell Orders

![Screen Shot 2021-03-23 at 21 05 12](https://user-images.githubusercontent.com/81108192/112218295-77644e00-8c1b-11eb-98ac-1ece48509c02.png)

## Last 30 days trade

Trade History                                                                                                       

![Screen Shot 2021-03-23 at 21 05 47](https://user-images.githubusercontent.com/81108192/112218383-8ba84b00-8c1b-11eb-94b9-e6e72029634e.png)

PNL Analysis

![Screen Shot 2021-03-23 at 21 06 24](https://user-images.githubusercontent.com/81108192/112218456-a1b60b80-8c1b-11eb-8d0a-17cf6f5185fb.png)

## Functions

- [x] Support multiple symbols
- [x] Remove unused methods - Bollinger Bands, MACD Stop Chaser
- [x] Support a maximum purchase amount per symbol
- [x] Develop backend to send cache values for frontend
- [x] Develop a simple frontend to see statistics
- [x] Fix the issue with the configuration
- [x] Update frontend to remove cache
- [x] Fix the issue with rounding when places an order
- [x] Fix the issue with persistent Redis
- [x] Fix the bug last buy price not removed
- [x] Update frontend to be exposed to the public using the local tunnel
- [x] Display account balances in the frontend
- [x] Update frontend to change symbols in the configuration
- [x] Update frontend to change last buy price per symbol
- [x] Change to more persistence database - MongoDB - for configuration and last
      buy price
- [x] Display estimated value in the frontend
- [x] Support other FIAT symbols such as BUSD, AUD
- [x] Allow entering more decimals for the last buy price
- [x] Override buy/sell configuration per symbol
- [x] Support PWA for frontend - **now support "Add to Home screen"**
- [x] Enable/Disable symbols trading, but continue to monitor
- [x] Add max-size for logging
- [x] Execute chaseStopLossLimitOrder on every process
- [x] Support buy trigger percentage
- [x] **Breaking changes** Re-organise configuration structures
- [x] Apply chase-stop-loss-limit order for buy signal as well
- [x] Added more candle periods - 1m, 3m and 5m
- [x] Allow to disable local tunnel
- [x] Fix the bug with handling open orders
- [x] Fix the bug with limit step in the frontend
- [x] Updated the frontend to display buy open orders with the buy signal
- [ ] Allow browser notification in the frontend
- [ ] Secure frontend with the password
- [ ] Allow to disable sorting in the frontend

## Contributing

Thanks all for your contributions...
    
![Screen Shot 2021-03-21 at 19 11 59](https://user-images.githubusercontent.com/81108192/111917690-519f4380-8a79-11eb-9d01-de457b1655f6.png)
    
ETH WALLET: 0xA1134858c168568CBE37649D16723eC8F782e0A2

![Screen Shot 2021-03-21 at 21 56 54](https://user-images.githubusercontent.com/81108192/111922186-5b807100-8a90-11eb-8504-a3fc3ae35052.png)

BTC WALLET: 3N928MmFq51kbf6fE3fxJbtggBhcjMAhSQ
