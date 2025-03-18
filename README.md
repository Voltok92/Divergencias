import backtrader as bt

class DivergenceStrategy(bt.Strategy):
    params = dict(
        ema_period=12,
        rsi_period=14,
        parabolic_sar_acc=0.02,
        parabolic_sar_max=0.2,
        profit_target=40
    )

    def __init__(self):
        self.ema = bt.indicators.ExponentialMovingAverage(period=self.params.ema_period)
        self.rsi = bt.indicators.RSI(period=self.params.rsi_period)
        self.parabolic_sar = bt.indicators.ParabolicSAR(acc=self.params.parabolic_sar_acc, max=self.params.parabolic_sar_max)
        self.trailing_stop = None
        self.entry_price = None

    def next(self):
        if self.position:
            # Trailing stop: si la ganancia es de 40 puntos, mover al cierre de las dos velas anteriores
            if self.data.close[0] >= self.entry_price + self.params.profit_target:
                self.trailing_stop = min(self.data.close[-1], self.data.close[-2])

            # Cerrar por trailing stop o divergencia contraria
            if (self.trailing_stop and self.data.close[0] <= self.trailing_stop) or self.detect_divergence(reverse=True):
                self.close()
                return

        # Entrada si hay divergencia y el precio cierra por encima de la EMA
        if self.detect_divergence() and self.data.close[-1] < self.ema[-1] and self.data.close[0] > self.ema[0]:
            self.buy()
            self.entry_price = self.data.close[0]
            self.stop_loss = self.parabolic_sar[0]  # Stop Loss en el SAR más alto

    def detect_divergence(self, reverse=False):
        """Detecta divergencias en el RSI comparando los picos con el precio"""
        if len(self.data) < 5:
            return False

        rsi_peaks = [self.rsi[-4], self.rsi[-2]]
        price_peaks = [self.data.high[-4], self.data.high[-2]]

        if reverse:
            return (rsi_peaks[0] < rsi_peaks[1]) and (price_peaks[0] > price_peaks[1])
        else:
            return (rsi_peaks[0] > rsi_peaks[1]) and (price_peaks[0] < price_peaks[1])

# Configuración del backtest
cerebro = bt.Cerebro()
data = bt.feeds.GenericCSVData(dataname='nasdaq_5min.csv', timeframe=bt.TimeFrame.Minutes, compression=5, headers=True)

cerebro.adddata(data)
cerebro.addstrategy(DivergenceStrategy)
cerebro.broker.set_cash(10000)
cerebro.run()
cerebro.plot()
