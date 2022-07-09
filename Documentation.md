# Advanced_Programming_Final_Project_GroupL -- Documentation

## Project: Personal saving management system

by GroupL:
Andrei Ivlev - Animesh Gautam - Arundhati Pillay - Jiayi Yang - Tsaichi Lee

### **Conceptual specifications**

This saving management system is used with N26 monthly balance sheet and has several functions:
1. it can show the user the total expense and income of the month
2. it can show the user how much they have spent in different spending categories
3. it can help user to start a saving plan

### **Technical Specifications**

Development language: **Python 3**

The system is designed with 2 classes: **Process** for pre-processing of data and **Budget** for all functions stored in **project3_GroupL.py**.

**Class functionality**:
1. The data-preprocessing class `Process`: 
- To clean the csv file downloaded from N26 account and make sure it can be can be used for later functions

2. The fucntion class `Budget`:
- To creat all the functions 


#### **`project3_GroupL.py`**
-create data-processing function to process original balance dataset downloaded from N26
```py
import pandas as pd
import numpy as np 

class Process:
    def __init__(self):
        df = pd.read_csv('statement-2022-06.csv')
        df.columns = ['Description', 'Date', 'Amount']

        #select the rows that contain the previous balance information
        previous_balance = df.loc[df['Description']=='Previous balance']['Date']

        #select the rows that describe the outgoing transaction
        outgoing_transactions = df.loc[df['Description']=='Outgoing transactions']['Date']

        #select the rows that descirbe the incoming transaction
        incoming_transactions = df.loc[df['Description']=='Incoming transactions']['Date']

        #select the rows that show the current balance
        new_balance = df.loc[df['Description']=='Your new balance']['Date']

        #get the row index of previous balance
        row = df[df['Description'] == 'Previous balance'].index.tolist()[0]

        #filter out the rows contain useless information
        df = df.iloc[:row]

        #drop out none values
        df = df[df['Description'].notna()]
        df = df.reset_index(drop=True)
        df = df.replace('Booking Date', np.nan)

        #get the indexes of the rows that contain useful transaction information
        first_list = df.index[df['Date'].notna() == True].tolist()
        second_list = [x+1 for x in first_list]
        index_list = first_list + second_list
        index_list = sorted(index_list)
        df = df.iloc[index_list]
        df = df.reset_index(drop=True)

        #change the form to get the number itself
        df['Description'] = np.where((df['Date'].isna() == True) & (df['Description'].str.contains(" • ", case=False) == False),
                                     np.nan,
                                     df['Description'])
        df['Description'] = np.where(df['Description'].str.contains(" • ", case=False) == True,
                                     df['Description'].str.split(' • ').str[1],
                                     df['Description'])
        df['Description'] = df['Description'].fillna('Other')

        #select the rows that contain category information
        df['Category'] = np.where(df['Date'].isna() == False,
                                  df['Description'].shift(-1),
                                  np.nan)
        df = df[df['Category'].notna()]
        df = df.reset_index(drop=True)

        #get the number of spending in the numeric form instead of string
        df['Amount'] = df['Amount'].str.replace(',', '.')
        df['Amount'] = df['Amount'].str.replace('€', '')
        df['Amount'] = pd.to_numeric(df['Amount'],errors = 'coerce')
        df['Category'] = np.where(df['Amount'] > 0, 
                                     'Income', 
                                     df['Category'])

        #get the income seperately 
        self.l_income = list(df[df['Category'] == 'Income']['Amount'])

        #select only spendings but not income
        df_cat = df[['Category', 'Amount']]
        df_cat = df_cat[df_cat['Category'] != 'Income']
        df_cat['Amount'] = abs(df_cat['Amount'])
        self.expense = df_cat.groupby('Category')['Amount'].apply(list).to_dict()
```
        
-New class to create all functions 

-Get total expenses and income
```py
class Budget(Process):
    def __init__(self):
        #self.expense = { 'Shopping' : [10,20], 'Other' : [30,40], 'Entertainment' : [50,60]}
        #self.l_income = [100,200,300]
        super().__init__()
        #get the total income and expense by suming they up and round them to 2 digits
        self.total_income = sum(self.l_income)
        self.total_income = round(self.total_income,2)
        self.total_expense = 0
        for x in self.expense.values():
            for y in x:
                self.total_expense += y
        self.total_expense = round(self.total_expense,2)

        #calculate the balance 
        self.balance = self.total_income - self.total_expense
        self.balance = round(self.balance,2)
```
    
-create display function to show the result
```py
    #create display function to show the result
    def display_income(self):
        print("\nTotal income: {}".format(self.total_income))
    
    def display_expense(self):
        print("\nTotal expenses: {}".format(self.total_expense))
    
    def user_balance(self):
        print("\nUser balance: {}".format(self.balance))
 ```
    
-get a report of the total expense of different categories 
```py
    def expense_distribution(self):
        for category,l in self.expense.items():
            cat_sum = 0 
            for i in l:
                cat_sum += i      
            print("\n{} : {} %".format(category, round((cat_sum / self.total_expense)*100,2)))
```

-create a function to query the expense of a certain category
```py
    def cat_expense(self):
        self.cat = {}
        for category,l in self.expense.items():
            cat_sum = 0 
            for i in l:
                cat_sum += i
            self.cat[category] = cat_sum
        cat_required = input('Please enter the category expense you want to query ')
        if cat_required in self.cat.keys():
            print(round(self.cat[cat_required],2))
        else:
            print('Sorry, there is no such category')
```

-create the fucntion for users' saving plan in amount of money
```py
    def savings_basic(self):
        #ask about the saving goal
        self.basic_goal = int(input('\nEnter amount you want to save over the next year: '))

        #calculate the saving goal per month
        self.basic_monthly_amount = self.basic_goal / 12

        #get the percentage that have to be saved from the income
        self.basic_percent = (self.basic_monthly_amount * 100) / self.total_income
        print("You must save {} {} of your income per month".format(round(self.basic_percent,2),'%'))
        print("i.e. {} / month".format(round(self.basic_monthly_amount,2)))

        #compare to see if the balance can meet the saving goal per month
        if(self.basic_monthly_amount > self.balance):
            print("Based on your balance, you need to reduce expenses or increase income to reach your savings goals")
        else:
            print("Based on your balance, you are on track to reach your savings goals")
```

-create a function to make a saving plan based in percentage of income
```py
    def savings_advanced(self):
        self.adv_percent = int(input('\nEnter % ' + 'of income you want to save: '))
        self.adv_monthly_amount = (self.adv_percent * self.total_income) / 100
        print("You must save {} / month".format(self.adv_monthly_amount))
        #compare to see if the balance can meet the saving goal per month
        if(self.adv_monthly_amount > self.balance):
            print("Based on your balance, you need to reduce expenses or increase income to reach your savings goals")
        else:
            print("Based on your balance, you are on track to reach your savings goals")
```

-we also created functions for those who don't have a balance sheet and want to input income and expense manually
```py
    def add_income(self):
        while True:
            income_amount = int(input('Enter income amount: '))
            self.l_income.append(income_amount)
            income_name = input('Enter source of income: ')
            self.l_income_name.append(income_name)
            result = input('Add income? (y/n): ')
            if result == 'n' or result == 'N':
                break
        self.total_income = sum(self.l_income)
        self.total_expense = sum(self.l_expense)
        self.balance = self.total_income - self.total_expense
    
    def add_expense(self):
        while True:
            expense_amount = int(input('Enter expense amount: '))
            self.l_expense.append(expense_amount)
            expense_name = input('Enter expense type: ')
            self.l_expense_name.append(expense_name)
            result = input('Add expense? (y/n): ')
            if result == 'n' or result =='N':
                break
        self.total_income = sum(self.l_income)
        self.total_expense = sum(self.l_expense)
        self.balance = self.total_income - self.total_expense
    
    def income_distribution(self):
        for i in range(len(self.l_income)):
            print("{} : {} %".format(self.l_income_name[i],round((self.l_income[i] / self.total_income)*100, 2)))
```
    
-the gate to call the function
```py
if __name__ == '__main__':
    
    b = Budget()
    while True:
        #offers choices to get corresponding function
        print("1. Display total income")
        print("2. Display total expenses")
        print("3. Display user balance")
        print("4. Category wise distribution of expenses")
        print("5. Query the expense of a certain category")
        print("6. Savings suggestion")
        print("7. Monthly savings plan")
        choice = int(input("\nSelect option: "))
        #call the corresponding fucntion according to the request
        if choice == 1:
            b.display_income()
        elif choice == 2:
            b.display_expense()
        elif choice == 3:
            b.user_balance()
        elif choice == 4:
            b.expense_distribution()
        elif choice == 5:
            b.cat_expense()
        elif choice == 6:
            b.savings_basic()
        elif choice == 7:
            b.savings_advanced()

        print()
        x = input("\nDo you want to continue? (y/n)")
        if x == 'N' or x == 'n':
            break
```


# Code Commitment
|Name | Commitment | SHA |
|---- | ---- | ---- |
|Andrei Ivlev | create data-processing function to process original balance dataset downloaded from N26 | f9638214c65d20d657b785ead110c8709da3e3f3 |
|Andrei Ivlev | select only useful rows | 4ea09ab1ca498f7c799049cb35597bb2216bdac2 |
|Andrei Ivlev | transform the information into a dictionary for later use |601853f3c668ea6ade061831fc4afcf5f6bdac39 |
|Arundhati Pillay | Create a class to perform main functions |88beab567b97f48a12cb43fcd28478add8424a16 |
|Arundhati Pillay | Create a function to display all income |3372d9251597ef72b4d6317ca9cbbf068bcc4225 |
|Tsaichi Lee | display expense and user balance |3372d9251597ef72b4d6317ca9cbbf068bcc4225 |
|Tsaichi Lee | get a report of the total expense of different categories |154befa554e587d9666ac710f2edc63225c310e8 |
|Jiayi Yang | add a function to query the expense of every category |8a07378ee4a7b6705a5888d95f746d359d3491b8 |
|Jiayi Yang | add a function to set up the saving goals of next year |a356aeb149bf2dd8b92fe50c567744aba99494fa |
|Animesh |  | |
|Animesh |  | |

