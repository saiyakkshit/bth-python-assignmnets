# Importing required libraries
from helper import create_plot
from sim_parameters import TRASITION_PROBS, HOLDING_TIMES
import datetime
import pandas as pd
import numpy as np

# Defining function to simulate COVID-19 timeseries data
def run(countries_csv_name, countries, start_date, end_date, sample_ratio):
    countries_data = pd.read_csv(countries_csv_name)
    
    # Selecting data of specific countries mentioned in the function argument
    specific = countries_data[countries_data["country"].isin(countries)]
    
    # Reducing population of each country by the given sample ratios
    specific['population'] = round(specific['population']//sample_ratio)
    
    # Creating age group-wise data for each country based on their population percentage
    specific[['less_5', '5_to_14', '15_to_24', '25_to_64', 'over_65']] = specific[['less_5', '5_to_14', '15_to_24', '25_to_64', 'over_65']].apply(lambda x: round((x*specific['population'])//100))
    
    # Creating an empty dataframe to store age group, person_id and country data
    specific_data_frame = pd.DataFrame(columns=["person_id", "age_group_name", "country"])
    
    # Iterating through each row of specific dataframe to create person-wise data for each age group
    for i in range(specific.shape[0]):
        age_groups = ["less_5", "5_to_14", "15_to_24", "25_to_64", "over_65"]
        for age_group in age_groups:
            # Adding person data for each individual of a particular age group and country
            for k in range(int(specific.iloc[i][age_group])):
                k = len(specific_data_frame)
                specific_data_frame.loc[k] = [k, age_group, specific.iloc[i]["country"]]
    
    # Creating an empty dataframe to store simulated COVID-19 data
    markov_data = pd.DataFrame(columns=["person_id", "age_group_name", "country","date", "state", "staying_days", "prev_state"])
    
    # Initializing lists to store values for different columns of markov_data dataframe
    person_id, age_group_name, country, date, state, staying_days, prev_state = [], [], [], [], [], [], []
    
    # Iterating through each row of specific_data_frame dataframe to simulate COVID-19 data for each individual
    for i in range(specific_data_frame.shape[0]):
        # Converting start_date and end_date to datetime format
        first,last = datetime.datetime.strptime(start_date, '%Y-%m-%d'), datetime.datetime.strptime(end_date, '%Y-%m-%d')
        
        # Calculating the number of days between start and end date
        d, count, initial_state, initial_prev_state = int(str(last-first).split()[0]), 0, "H" , "H"
        
        # Simulating COVID-19 data for each individual for each day between start and end date
        for k in range(0, d+1):
            # Retrieving staying time for a particular age group and state
            hold,first = HOLDING_TIMES.get(specific_data_frame.iloc[i]["age_group_name"]).get(initial_state),pd.to_datetime(first) + pd.Timedelta(days=1)
            # get the holding time and calculate the next day based on the initial day
            person_id.append(specific_data_frame.iloc[i]['person_id']);age_group_name.append(specific_data_frame.iloc[i]["age_group_name"]);country.append(specific_data_frame.iloc[i]["country"]);date.append(str(first).split()[0]);state.append(initial_state);prev_state.append(initial_prev_state);staying_days.append(hold)
            # append values to lists
            initial_prev_state = initial_state 
            count += 1
            # increment the count
            if hold in [count, 0]:
            # check if the hold value is in the list of [count, 0]
                t = TRASITION_PROBS[specific_data_frame.iloc[i]["age_group_name"]][initial_state]
                # get the transition probability for the specific age group and initial state
                next_move = np.random.choice(list(t.keys()), p=list(t.values()))
                # randomly choose the next move based on the transition probabilities
                initial_prev_state, initial_state,count=initial_state,next_move,0
                # update the initial previous state, initial state, and count
            k += 1
            #increment value for k
        initial_state = 'H'
        # update the initial state to "H"
    markov_data = markov_data.assign(person_id=person_id, age_group_name=age_group_name, country=country, date=date, state=state, staying_days=staying_days, prev_state=prev_state)
    # assign the values to the Markov data
    markov_data.to_csv('a3-covid-simulated-timeseries.csv',index=False)
    # write the Markov data to a CSV file
    markov_data_frame = pd.read_csv('a3-covid-simulated-timeseries.csv')
    # read the Markov data from the CSV file
    group = markov_data_frame.groupby(['country', 'date', 'state'])['state'].count().reset_index(name='count')
    group_column = pd.crosstab(index=[group['date'], group['country']], columns=group['state'], values=group['count'], aggfunc='sum')
    # create a cross-tabulation table for the groups
    group_column = group_column.apply(lambda x: x.fillna(0)).astype(int)
    # fill any NaN values with 0 and convert the table to integer values
    group_column.to_csv('a3-covid-summary-timeseries.csv')
    # write the group data to a CSV file
    create_plot('a3-covid-summary-timeseries.csv',countries)