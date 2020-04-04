# Stock-correlation-matrix

## About
This program is written in Python and is made for calculating and visualizing correlations between stocks in a given index (i.e. S&P 500), which is useful for getting an idea of the relationships there are between certain stocks. Potentially this program could be used for selecting stocks for a portfolio that reduces risk. 

The idea behind this is of course the diversification strategy; by investing in a bunch of companies with zero (or little) correlation over time your portfolio will be more diverse, which in turn will minimize your risk. 

## How it works
The program grabs all the tickers (symbols) in an index from Wikipedia by extracting data from HTML using the Beautiful Soup package (web scraping). Then it turns the extrated data into to a beautifulsoup objct and creates a pickle.
```
def save_sp500_tickers():
    resp = requests.get('http://en.wikipedia.org/wiki/List_of_S%26P_500_companies')
    soup = bs.BeautifulSoup(resp.text, 'lxml')
    table = soup.find('table', {'class': 'wikitable sortable'})
    tickers = []
    for row in table.findAll('tr')[1:]:
        ticker = row.findAll('td')[0].text
        for rep in (('.','-'),('\n', '')):
            ticker = ticker.replace(*rep)
        tickers.append(ticker)
    # Getting price data for the total S&P 500
    tickers.append('^GSPC')
        # 'wb' stands for write binary
    with open("sp500tickers.pickle","wb") as f:
        pickle.dump(tickers,f)
    print(tickers)
    return tickers

save_sp500_tickers()
```
Using this pickle, the program calls the Yahoo Finance server to import price data for the tickers in the index. From here it saves the price data as individual csv-files (it can be a quite slow process so this just save the import in case the connection to the server breaks), for subsequently to compile them all into one dataframe. 
```
def get_data_from_yahoo(reload_sp500=False):
    if reload_sp500:
        tickers = save_sp500_tickers()
    else:
        with open("sp500tickers.pickle", "rb") as f:
            tickers = pickle.load(f)
    # Check if the folder 'stock_data' exists. If not create a new directory/folder.
    if not os.path.exists('stock_data'):
        os.makedirs('stock_data')

    start = dt.datetime(2015, 1, 1)
    end = dt.datetime(2020, 2, 1)

    for ticker in tickers:
        print(ticker)
        # Just in case the connection breaks, this saves the progress.
        if not os.path.exists('stock_data/{}.csv'.format(ticker)):
            df = web.DataReader(ticker, 'yahoo', start, end)
            df.to_csv('stock_data/{}.csv'.format(ticker))
        else:
            print('Already have {}'.format(ticker))

get_data_from_yahoo()

def compile_data():
    with open('sp500tickers.pickle', 'rb') as f:
        tickers = pickle.load(f)

    main_df = pd.DataFrame()

    for count, ticker in enumerate(tickers):
        df = pd.read_csv('stock_data/{}.csv'.format(ticker))
        df.set_index('Date', inplace = True)

        df.rename(columns = {'Adj Close': ticker}, inplace = True)
        df.drop(['Open','High','Low','Close','Volume'], 1, inplace = True)

        if main_df.empty:
            main_df = df
        else:
            main_df = main_df.join(df, how='outer')

        # Just to see how far we have come
        if count % 10 == 0:
            print(count)

    print(main_df.head())
    main_df.to_csv('sp500_joined_closes.csv')

compile_data()
```
Having the dataframe which consists of price data for the entire index, the program cauculates the correlation coefficients between all the stocks in the index. It then plots the correlations in a heatmap for a nice visual representation. 
```
def correlation_tabel():
    df = pd.read_csv('sp500_joined_closes.csv')
    df_corr = df.corr()
    #print(df_corr.head)

    # The correlation between S&P 500 and a single company
    company = 'AAPL'
    df_single_corr = df['^GSPC'].corr(df[company])
    #print('The correlation coefficient between S&P500 and ', company, ': ' + '{:.3f}'.format(df_single_corr), sep='')

    data = df_corr.values
    fig = plt.figure()
    ax = fig.add_subplot(1, 1, 1)

    heatmap = ax.pcolor(data, cmap = plt.cm.RdYlGn)
    fig.colorbar(heatmap)
    ax.set_xticks(np.arange(data.shape[0]) + 0.5, minor = False)
    ax.set_yticks(np.arange(data.shape[1]) + 0.5, minor = False)
    ax.invert_yaxis()
    ax.xaxis.tick_top()

    column_labels = df_corr.columns
    row_labels = df_corr.index

    ax.set_xticklabels(column_labels)
    ax.set_yticklabels(row_labels)
    plt.xticks(rotation = 90)
    heatmap.set_clim(-1, 1)
    plt.tight_layout()
    plt.show()

correlation_tabel()
```
