# Read rows from a file for inserting into plant-monitor table
with open("plants_data.txt", encoding='UTF-8') as data_file:
    data = data_file.readlines()
