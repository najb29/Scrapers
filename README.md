# Scrapers
import csv
import datetime
import re
import time
from bs4 import BeautifulSoup
import undetected_chromedriver as uc
from selenium.webdriver import ChromeOptions
from selenium.webdriver.chrome.service import Service
import pandas as pd
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.by import By


def address_insights(wallet_address, driver, current_time):

    driver.get(f'https://bitinfocharts.com/bitcoin/address/{wallet_address}')
    tries = 0
    while True:
        try:
            WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.ID, 'table_maina')))
            wallet_soup = BeautifulSoup(driver.page_source, features="html.parser")
            wait_ = bool(re.search('Checking', str(wallet_soup)))

            while wait_:
                print('Waiting...')
                time.sleep(10)
                driver.get(f'https://bitinfocharts.com/bitcoin/address/{wallet_address}')
                WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.ID, 'table_maina')))
                wallet_soup = BeautifulSoup(driver.page_source, features="html.parser")
                wait_ = bool(re.search('Checking', str(wallet_soup)))

            date_time = current_time
            last_24 = []
            last_week = []
            last_month = []
            final_24 = []
            final_week = []
            final_month = []

            for row_ in wallet_soup.find('table', {'class': 'table table-striped table-condensed'}).find_all('tr')[1:]:
                if 'Received' in str(row_.find_all('td')[0].text).strip():
                    data['total_btc_received'] = str(row_.find_all('td')[1].text).strip().split(' (')[0]
                    data['first_day_received'] = str(row_.find_all('td')[2].text).strip().split('first: ')[1]
                    data['ins'] = str(row_.find_all('td')[1].text).strip().split(' (')[1].split(' ins)')[0]
                if 'Sent' in str(row_.find_all('td')[0].text).strip():
                    data['total_btc_sent'] = str(row_.find_all('td')[1].text).strip().split(' (')[0]
                    data['outs'] = str(row_.find_all('td')[1].text).strip().split(' (')[1].split(' outs)')[0]
                if 'Unspent' in str(row_.find_all('td')[0].text).strip():
                    data['unspent_output'] = str(row_.find_all('td')[0].text).strip().split('outputs: ')[1]

            for transaction in wallet_soup.find('table', {'id': 'table_maina'}).find('tbody').find_all('tr'):
                if (datetime.datetime.strptime(date_time, '%Y-%m-%d %H:%M:%S') - datetime.datetime.strptime(str(transaction.find_all('td')[1].text).strip(),'%Y-%m-%d %H:%M:%S')).days == 0:
                    last_24.append(str(transaction.find_all('td')[2].text).strip().split(' (')[0])
                if (datetime.datetime.strptime(date_time, '%Y-%m-%d %H:%M:%S') - datetime.datetime.strptime(str(transaction.find_all('td')[1].text).strip(),'%Y-%m-%d %H:%M:%S')).days <= 7:
                    last_week.append(str(transaction.find_all('td')[2].text).strip().split(' (')[0])
                if (datetime.datetime.strptime(date_time, '%Y-%m-%d %H:%M:%S') - datetime.datetime.strptime(str(transaction.find_all('td')[1].text).strip(),'%Y-%m-%d %H:%M:%S')).days <= 30:
                    last_month.append(str(transaction.find_all('td')[2].text).strip().split(' (')[0])

            data['7_transactions'] = len(last_week)
            data['30_transactions'] = len(last_month)
            data['24_transactions'] = len(last_24)

            if last_24:
                for item in last_24:
                    newstr = ''.join((ch.replace(',', '') if ch in '0123456789.,-' else ' ') for ch in item)
                    for i in newstr.split():
                        final_24.append(float(i))
                data['24_net_change'] = str(sum(final_24)) + ' BTC'

            if last_week:
                for item in last_week:
                    newstr = ''.join((ch.replace(',', '') if ch in '0123456789.,-' else ' ') for ch in item)
                    for i in newstr.split():
                        final_week.append(float(i))
                data['7_net_change'] = str(sum(final_week)) + ' BTC'

            if last_month:
                for item in last_month:
                    newstr = ''.join((ch.replace(',', '') if ch in '0123456789.,-' else ' ') for ch in item)
                    for i in newstr.split():
                        final_month.append(float(i))
                data['30_net_change'] = str(sum(final_month)) + ' BTC'

            with open('top_3000.csv', 'a', newline='', encoding='utf-8-sig') as outputFile:
                fieldnames = ['RANK', 'ADDRESS', 'BTC AMOUNT', f'MARKET VALUE @[{current_timestamp}]', 'Total BTC Received', 'Total BTC Sent', 'First Day Received', 'Number of Ins', 'Number of Outs', 'Unspent Output', '24 HOUR NET CHANGE', '24 HOUR NUMBER OF TRANSACTIONS', '7 DAY NET CHANGE', '7 DAY NUMBER OF TRANSACTIONS', '30 DAY NET CHANGE', '30 DAY NUMBER OF TRANSACTIONS']
                writer = csv.DictWriter(outputFile, fieldnames=fieldnames)
                writer.writerow({
                    'RANK': data['rank'],
                    'ADDRESS': data['address'],
                    'BTC AMOUNT': data['btc_amount'],
                    f'MARKET VALUE @[{current_time}]': data['market_value'],
                    '24 HOUR NET CHANGE': data['24_net_change'],
                    '24 HOUR NUMBER OF TRANSACTIONS': data['24_transactions'],
                    '7 DAY NET CHANGE': data['7_net_change'],
                    '7 DAY NUMBER OF TRANSACTIONS': data['7_transactions'],
                    '30 DAY NET CHANGE': data['30_net_change'],
                    '30 DAY NUMBER OF TRANSACTIONS': data['30_transactions'],
                    'Total BTC Received': data['total_btc_received'],
                    'Total BTC Sent': data['total_btc_sent'],
                    'First Day Received': data['first_day_received'],
                    'Number of Ins': data['ins'],
                    'Number of Outs': data['outs'],
                    'Unspent Output': data['unspent_output']
                })
            break
        except:
            if tries < 3:
                driver.get(f'https://bitinfocharts.com/bitcoin/address/{wallet_address}')
                tries += 1
                continue
            else:
                print(f'Error accessing https://bitinfocharts.com/bitcoin/address/{wallet_address}')
                break



if __name__ == "__main__":
    current_timestamp = str(datetime.datetime.now()).split('.')[0]
    with open('top_3000.csv', 'w', newline='', encoding='utf-8-sig') as outputFile:
        fieldnames = ['RANK', 'ADDRESS', 'BTC AMOUNT', f'MARKET VALUE @[{current_timestamp}]', 'Total BTC Received', 'Total BTC Sent', 'First Day Received', 'Number of Ins', 'Number of Outs', 'Unspent Output', '24 HOUR NET CHANGE', '24 HOUR NUMBER OF TRANSACTIONS', '7 DAY NET CHANGE', '7 DAY NUMBER OF TRANSACTIONS', '30 DAY NET CHANGE', '30 DAY NUMBER OF TRANSACTIONS']
        writer = csv.DictWriter(outputFile, fieldnames=fieldnames)
        writer.writeheader()

    options = ChromeOptions()
    options.add_argument("--disable-blink-features")
    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_argument("--disable-site-isolation-trials")
    options.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36")
    driverService = Service('./chromedriver')
    capabilities = DesiredCapabilities().CHROME
    capabilities['pageLoadStrategy'] = "eager"
    driver = uc.Chrome(options=options, capabilities=capabilities)

    headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/103.0.0.0 Safari/537.36'}

    for page in range(10):

        driver.get(f'https://bitinfocharts.com/top-100-richest-bitcoin-addresses-{page + 1}.html')
        WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.ID, 'tblOne')))
        soup = BeautifulSoup(driver.page_source, features="html.parser")

        for wallet in soup.find('table', {'id': 'tblOne'}).find('tbody').find_all('tr'):
            print(f'Processing wallet rank #{str(wallet.find_all("td")[0].text).strip()}')

            data = {
                'address': str(wallet.find_all('td')[1].find('a').text).replace('.', ''),
                'rank': str(wallet.find_all('td')[0].text).strip(),
                'btc_amount': str(wallet.find_all('td')[2].text).strip().split(' BTC (')[0],
                'market_value': str(wallet.find_all('td')[2].text).strip().split('(')[1].split(')')[0],
                '7_net_change': '',
                '30_net_change': '',
                '7_transactions': 0,
                '30_transactions': 0,
                '24_net_change': '',
                '24_transactions': 0,
                'total_btc_received': 'N/A',
                'total_btc_sent': 'N/A',
                'first_day_received': 'N/A',
                'ins': 'N/A',
                'outs': 'N/A',
                'unspent_output': 'N/A'
                }

            address_insights(data['address'], driver, current_timestamp)

        for wallet in soup.find('table', {'id': 'tblOne2'}).find_all('tr'):
            print(f'Processing wallet rank #{str(wallet.find_all("td")[0].text).strip()}')

            data = {
                'address': str(wallet.find_all('td')[1].find('a').text).replace('.', ''),
                'rank': str(wallet.find_all('td')[0].text).strip(),
                'btc_amount': str(wallet.find_all('td')[2].text).strip().split(' BTC (')[0],
                'market_value': str(wallet.find_all('td')[2].text).strip().split('(')[1].split(')')[0],
                '7_net_change': '',
                '30_net_change': '',
                '7_transactions': 0,
                '30_transactions': 0,
                '24_net_change': '',
                '24_transactions': 0,
                'total_btc_received': 'N/A',
                'total_btc_sent': 'N/A',
                'first_day_received': 'N/A',
                'ins': 'N/A',
                'outs': 'N/A',
                'unspent_output': 'N/A'
                }

            address_insights(data['address'], driver, current_timestamp)

    print('Analyzing the corresponding wallets')
    result = pd.read_csv('top_3000.csv')

    for value in result.values.tolist():
        if str(value[10]) != 'nan':
            for index, row in result.iterrows():
                if str(result.loc[index, '24 HOUR NET CHANGE']) != 'nan' and result.loc[index, '24 HOUR NUMBER OF TRANSACTIONS'] == value[11]:
                    if float(str(result.loc[index, '24 HOUR NET CHANGE']).split(' BTC')[0]) != 0 and float(str(result.loc[index, '24 HOUR NET CHANGE']).split(' BTC')[0]) == -1 * float(str(value[10]).split(' BTC')[0]):
                        result.loc[index, '24 Hour Corresponding Wallet'] = str(value[0]) + '. ' + str(value[1])

        if str(value[12]) != 'nan':
            for index, row in result.iterrows():
                if str(result.loc[index, '7 DAY NET CHANGE']) != 'nan' and result.loc[index, '7 DAY NUMBER OF TRANSACTIONS'] == value[13]:
                    if float(str(result.loc[index, '7 DAY NET CHANGE']).split(' BTC')[0]) != 0 and float(str(result.loc[index, '7 DAY NET CHANGE']).split(' BTC')[0]) == -1 * float(str(value[12]).split(' BTC')[0]):
                        result.loc[index, '7 Day Corresponding Wallet'] = str(value[0]) + '. ' + str(value[1])

        if str(value[14]) != 'nan':
            for index, row in result.iterrows():
                if str(result.loc[index, '30 DAY NET CHANGE']) != 'nan' and result.loc[index, '30 DAY NUMBER OF TRANSACTIONS'] == value[15]:
                    if float(str(result.loc[index, '30 DAY NET CHANGE']).split(' BTC')[0]) != 0 and float(str(result.loc[index, '30 DAY NET CHANGE']).split(' BTC')[0]) == -1 * float(str(value[14]).split(' BTC')[0]):
                        result.loc[index, '30 Day Corresponding Wallet'] = str(value[0]) + '. ' + str(value[1])

    result.to_csv('top_3000.csv', index=False)
