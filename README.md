# Paratica Task 1
 RSI indicator based REST service.
 
# Task
"
Your service will have 4 REST endpoints :
1. Implement GET /history endpoint for returning previous signals on whole pairs,
which generated by step 4
2. Implement GET /history/<parity_symbol> endpoint for returning previous signals
on parity_symbol == <parity_symbol> which generated by step 4
3. Implement GET /check_signals/<parity_symbol> endpoint that simultaneously checks the signals created on the parity with the parity_symbol ==
<parity_symbol> and showing results to users without saving to the database
4. Implement PUT /save_signals/<parity_symbol> endpoint that first checks and then saves the signals created on the parity with the parity_symbol ==
<parity_symbol>
 "
 
 # Requirements 
On this task a database system and an api system required. For the sake of simplicity I recommend FastAPI for api service and Postgres Database for database.
You can check the implementations online and install easily. You can use the pipfile or pipfile.lock to install required libraries 
run this command for dependencies -> pipenv install or pipenv install --dev
-> https://www.postgresql.org/download/
-> https://fastapi.tiangolo.com

# Creating Database
After installing postgres and run your database, you can create a simple py file to create your table. You can check the CreateTableDb.py file for this purpose.

connecting database: 

con = psycopg2.connect(database="postgres", user="postgres", password=postgres_pass, host="127.0.0.1", port="5432")

creating table:

cur = con.cursor()

cur.execute('''CREATE TABLE RSI_SIGNAL
      (id               SERIAL      PRIMARY KEY,
      pair_symbol       CHAR(50)    NOT NULL,
      signal_date       TIMESTAMP   NOT NULL,
      rsi_2             real        NOT NULL,
      rsi_1             real        NOT NULL,
      previous_candle   real        NOT NULL);''')
      
con.commit()

con.close()

Also I create a file named AddToDb.py and create a function to easily add a record to database. This function just connects to database and run a INSERT INTO sql command with given parameters. 

 def addRecord(pair_symbol, signal_date, rsi_2, rsi_1, previous_candle):
 
For this task only one table required to hold informations:
● pair_symbol

● signal_date

● RSI[-2] value

● RSI[-1] value

● close value of the previous candle (Candle[-1])

You can easily run SQL commands with python using psycopg2 library -> https://www.psycopg.org/docs/usage.html
But not the best choice for sure, normally best method is the use ORM methods with an object oriented language, in our case python.
Here is the useful documentation using sqlalchemy with fast api -> https://fastapi.tiangolo.com/tutorial/sql-databases/

# What is RSI?
"The relative strength index (RSI) is a momentum indicator used in technical analysis that measures the magnitude of recent price changes to evaluate overbought or oversold conditions in the price of a stock or other asset." it may look confusing but simply its a indicator that shows if the given currency is overbought or oversold, so that we can make a strategy on buying or selling and it is calculated with this formula:

<img width="583" alt="Screen Shot 2022-01-17 at 01 42 55" src="https://user-images.githubusercontent.com/94080241/149681094-3b8967fb-262a-40ad-909e-79947e3c4a55.png">

If curious you can check the link for more information -> https://www.investopedia.com/terms/r/rsi.asp

But numpy and ta-lib libraries are providing easy functions to calculate it. 

        np_closes = numpy.array([item[0] for item in closes_at])
        rsi = talib.RSI(np_closes, w_length)
      
Here we need a window length (w_length) to determine how many days are gonna be used to calculate average gain-loss (if not given set to = 14). After setting window length all we need to do is making a numpy array from our close price list and using RSI function to calculate RSI values. This function will return a list again but for first 14 (number of window length) element will be None since our data will be complete after 14 price and our average will be calculated. So we on the 15th data we can calculate a RSI value.

# Signal logic
Buying strategy is built on the RSI indicator and checks the lastly closed candle(RSI[-1]) and the one closed before the last one(RSI[-2]). The condition that the value of RSI[-2] was lower than 40 and the value of RSI[-1] is higher than 40 is proper for buying strategy. If any condition like this occurs in a given parity symbol’s Binance Exchange history, that should be presented to the user. (Only the 4H candles should be used for the task)

# Endpoints  webs

# 1- GET /history
After creating our database and running our simple fast api and unvicorn based app, we can now create our first endpoint. This endpoint will get all signal records saved before from the database and send to user as a response.

With this command:

sql_command = "SELECT pair_symbol, signal_date, rsi_2, rsi_1, previous_candle from RSI_SIGNAL"

we can easily select all data from our database. To run use cursor as shown before. After we can use the cursor.fetchall() function to fetch all data to an array and we can return this array to the user.

# 2- GET /history/<parity_symbol>
Only difference is we should only get the records with given parity_symbol (example: BTCUSDT). 

sql_command = f"SELECT pair_symbol, signal_date, rsi_2, rsi_1, previous_candle from RSI_SIGNAL where pair_symbol=\'{parity_symbol}\'"

We can use WHERE command to filter data and send to user back as same before.

# 3- GET /check_signals/<parity_symbol>
This is the part where things get tricky for me. First thing I thought was this part will be always working and checking signals non-stop but its not and it should check whenever a request came. But I tried something that after taking request a thread is created with checkRSI (check checkRSI.py) function and connecting to Binance's web-socket then retrieve live data for given parity_symbol. Normally another API should be used this purpose but since this is my first task I tried something different. I used threading to let main.py work and take care of requests while other thread check live data from Binance and looking for a situation that we can create a signal. Also last 14 (w_length) price data to populate closed_prices list so we can calculate RSI for last price data. After thread starts running if there is any creatable signal it hold on created_signals list (this list will be used on next endpoint). If another request came then we can return back our created signals if there is any.

# 4- PUT /save_signal/<parity_symbol>
Finally if we get this request. We first check created_signals and look if there is any signal created for given parity symbol. If not we send a error response saying there is no signals to save. If there is any we saved them to database using addToDb() and return success message
