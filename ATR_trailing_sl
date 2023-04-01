import time
import numpy as np
import pandas as pd
import MetaTrader5 as mt5



# Initialize the connection
mt5.initialize()
# Add a helper function to calculate the ATR

def calculate_atr(symbol, timeframe, period):
    
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, period + 1)
    df = pd.DataFrame(rates, columns=["time", "open", "high", "low", "close", "tick_volume", "spread", "real_volume"])
    
    # Calculate True Range
    df["tr0"] = abs(df["high"] - df["low"])
    df["tr1"] = abs(df["high"] - df["close"].shift())
    df["tr2"] = abs(df["low"] - df["close"].shift())
    df["tr"] = df[["tr0", "tr1", "tr2"]].max(axis=1)
    
    if len(df) < period:
        return np.nan

    # Calculate ATR
    atr = df["tr"].rolling(period).mean().iloc[-1]

    return atr


def create_sl_order(ticket, symbol, sl_price, sl_type):
    request = {
        "action": mt5.TRADE_ACTION_SLTP,
        "symbol": symbol,
        "position": ticket,
        "sl": sl_price,
        "type": sl_type,
        "magic": 0,
        "comment": "SL order",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_RETURN,
    }
    result = mt5.order_send(request)
    if result.retcode is not None and result.retcode != mt5.TRADE_RETCODE_DONE:
        print(f"Failed to create SL order for ticket {ticket}, error code: {result.retcode}")
    else:
        print(f"SL order created for ticket {symbol}, {time}, sl price {sl_price}")

def modify_sl_order(ticket, symbol, sl_price, sl_type):
    sl_order = mt5.orders_get(symbol=symbol, magic=0, comment="SL order")
    if sl_order is not None and len(sl_order) > 0:
        order = sl_order[0]
        if order.price != sl_price:
            request = {
                "action": mt5.TRADE_ACTION_SLTP,
                "symbol": symbol,
                "position": ticket,
                "order": order.ticket,
                "sl": sl_price,
                "type": sl_type,
                "magic": 0,
                "comment": "SL order",
                "type_time": mt5.ORDER_TIME_GTC,
                "type_filling": mt5.ORDER_FILLING_RETURN,
            }
            result = mt5.order_send(request)
            if result.retcode != mt5.TRADE_RETCODE_DONE:
                print(f"Failed to modify SL order for ticket {ticket}, error code: {result.retcode}")
            else:
                print(f"SL order modified for ticket {symbol}, {time}, sl price {sl_price}")
    else:
        create_sl_order(ticket, symbol, sl_price, sl_type)




def main():
    print("Starting the trailing stop loss script...")
    update_interval_seconds = 5

    try:
        while True:
            positions = mt5.positions_get()
            if positions is not None and len(positions) > 0:
                print(f"{len(positions)} open position(s) found.")
                for position in positions:
                    try:
                        symbol = position.symbol
                        ticket = position.ticket
                        volume = position.volume
                        current_sl = position.sl
                        open_price = position.price_open
                        current_price= position.price_current
                        symbol_info = mt5.symbol_info(symbol) 
                        if symbol_info is None: 
                            print(symbol, "symbol not found") 
                        else: 
                            point = mt5.symbol_info(symbol).point
                        spread_adjustment= symbol_info.spread * point

                        print(f" info updated succesfully for symbol {symbol}")


                        print(f"position info {symbol}, volume {volume}, open price {open_price}, current SL {current_sl},current price {position.price_current}, current spread {symbol_info.spread}")

                        # Set ATR parameters
                        atr_period = 14
                        atr_timeframe = mt5.TIMEFRAME_M1
                        
                        atr_value = calculate_atr(symbol, atr_timeframe, atr_period) # calculate ATR
                        trail_activation= 1 * atr_value 
                        trail_step = 0.2 * atr_value  # set ATR percentage
                        trail_distance= 0.5 * atr_value

                        sl_activation_long = open_price + trail_activation + spread_adjustment
                        sl_activation_short = open_price - (trail_activation + spread_adjustment)
                        print(f"current_atr {round(atr_value,5)} current spread {+ symbol_info.spread}")

                    

                        # Check if current price has crossed the new stop loss value
                        if position.type == mt5.POSITION_TYPE_BUY and current_price is not None and current_price < (sl_activation_long):
                            message = f"{symbol} LONG\nentry price: {open_price}\ncurrent ATR: {atr_value}\n price needed for SL to start trailing: {round(sl_activation_long,5)}"
                            print(message)  # print message to console for verification
                        elif position.type == mt5.POSITION_TYPE_SELL and current_price is not None and current_price > (sl_activation_short):
                            message = f"{symbol} SHORT\nentry price: {open_price}\ncurrent ATR: {atr_value}\n price needed for SL to start trailing: {round(sl_activation_short,5)}"
                            print(message)
                        
                        # Create initial stop loss if it doesn't exist and set it in profit
                        if current_sl == 0:
                            if position.type == mt5.POSITION_TYPE_BUY and current_price > sl_activation_long:
                                initial_sl = current_price - trail_distance
                                create_sl_order(ticket, symbol, initial_sl, mt5.ORDER_TYPE_BUY)
                            elif position.type == mt5.POSITION_TYPE_SELL and current_price < sl_activation_short:
                                initial_sl = current_price + trail_distance
                                create_sl_order(ticket, symbol, initial_sl, mt5.ORDER_TYPE_SELL)

                        # Trailing stop loss for a buy order
                        if position.type == mt5.POSITION_TYPE_BUY:
                            profit = (current_price - open_price) 
                            if profit > atr_value:
                                new_sl = current_price - trail_distance
                                if (current_price - new_sl) >= trail_step and (new_sl > current_sl):
                                    # Modify stop loss
                                    modify_sl_order(ticket, symbol, new_sl, mt5.ORDER_TYPE_BUY)

                        # Trailing stop loss for a sell order
                        elif position.type == mt5.POSITION_TYPE_SELL:
                            profit = (open_price - current_price) 
                            if profit > atr_value:
                                new_sl = current_price + trail_distance
                                if (new_sl - current_price) >= trail_step and (new_sl < current_sl):
                                    # Modify stop loss
                                    modify_sl_order(ticket, symbol, new_sl, mt5.ORDER_TYPE_SELL)
                    except Exception as e:
                            print(f"Error processing position {symbol}:{e}")
            else:
                print("No open positions found.")


            time.sleep(update_interval_seconds)
    except KeyboardInterrupt:
        print("Exiting...")

if __name__ == "__main__":
    main()
# Deinitialize the connection to the trading platform
mt5.shutdown()
