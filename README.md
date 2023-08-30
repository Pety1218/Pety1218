import os
print("Starting the program...")
import tkinter as tk
from tkinter import filedialog, Label, Button
import pandas as pd
from pandasgui import show

# Define the driver-company mapping
driver_company_map = {
    "Apovl Adimir": "Fast Express",
    "Gentea Florin": "Fast Express",
    "Matei Denislav": "Fast Express",
    "MILEN Atanas": "Fast Express",
    "Muca Florentin": "Fast Express",
    "Jurubita Razvan": "Fast Express",
    "Flegonov Christian": "Daniel Ontheroad",
    "Petcana Cristina": "Daniel Ontheroad",
    "Dimitrov F": "Daniel Ontheroad",
    "Daniel Stoianov": "DE Cargo Speed",
    "Leuce Petre": "DE Cargo Speed",
    "Nikolay Bogachev": "Stef Trans",
    "Valentin bogatinov": "Stef Trans",
    "alexandru bogdan": "Bis General",
    "Sorinescu I.": "Bis General"
}
companies = list(set(driver_company_map.values()))

# Global variables to store data
trip_df = None
invoice7_df = None
invoice30_df = None
result_data = {}


def add_totals_to_dataframe(df):
    totals_row = {
        'Trip ID': 'Total',
        'Driver': '',
        'Plata la 7 zile': df['Plata la 7 zile'].sum(),
        'Plata la 30 zile': df['Plata la 30 zile'].sum()
    }
    df.loc[len(df)] = totals_row
    return df

def calculeaza_comision(nume_companie, plata_totala):
    if nume_companie == "Fast Express":
        rata_comision = 0.01
    elif nume_companie == "Bis General":
        rata_comision = 0.05
    elif nume_companie in ["Stef Trans", "Daniel Ontheroad", "DE Cargo Speed"]:
        rata_comision = 0.015  # sau orice altă rată dorită pentru aceste companii
    else:
        rata_comision = 0.015
    return plata_totala * rata_comision 

def load_trip():
    global trip_df
    file_path = filedialog.askopenfilename(filetypes=[('CSV Files', '*.csv')])
    if not file_path:
        return
    trip_df = pd.read_csv(file_path)


def load_invoice7():
    global invoice7_df
    file_path = filedialog.askopenfilename(filetypes=[('Excel Files', '*.xlsx')])
    if not file_path:
        return
    invoice7_df = pd.read_excel(file_path)


def load_invoice30():
    global invoice30_df
    file_path = filedialog.askopenfilename(filetypes=[('Excel Files', '*.xlsx')])
    if not file_path:
        return
    invoice30_df = pd.read_excel(file_path)


def get_mapped_drivers(drivers):
    drivers = drivers.split(',')
    mapped_drivers = [driver for driver in drivers if driver in driver_company_map]
    return mapped_drivers


def process_data_for_company(company_name):
    if trip_df is None or invoice7_df is None or invoice30_df is None:
        print("Vă rugăm să încărcați mai întâi toate fișierele necesare.")
        return

    trip_df['Driver'].fillna("", inplace=True)

    # Filtrăm șoferii după companie
    company_drivers = [driver for driver, company in driver_company_map.items() if company == company_name]
    company_trip_df = trip_df[trip_df['Driver'].apply(lambda x: any(driver in company_drivers for driver in get_mapped_drivers(x)))]

    # Unim datele
    result = pd.merge(company_trip_df[['Trip ID', 'Driver']], invoice7_df[['Load ID', 'Gross Pay Amt']], left_on='Trip ID', right_on='Load ID', how='left')
    result = pd.merge(result, invoice30_df[['Load ID', 'Gross Pay Amt']], left_on='Trip ID', right_on='Load ID', how='left', suffixes=('_7days', '_30days'))

    # Redenumim coloanele
    result = result[['Trip ID', 'Driver', 'Gross Pay Amt_7days', 'Gross Pay Amt_30days']]
    result.columns = ['Trip ID', 'Driver', 'Plata la 7 zile', 'Plata la 30 zile']

    # Adăugăm rândul cu totaluri
    result = add_totals_to_dataframe(result)

    # Calculăm și adăugăm coloana cu comisioane
    result['Comision'] = result.apply(lambda row: calculeaza_comision(company_name, row['Plata la 7 zile'] + row['Plata la 30 zile']), axis=1)

    # Stocăm rezultatul pentru export
    result_data[company_name] = result

    # Afișăm rezultatele în GUI
    show(result[['Trip ID', 'Driver', 'Plata la 7 zile', 'Plata la 30 zile', 'Comision']])

def export_all_data():
    if not result_data:
        print("Nu există date de exportat.")
        return

    root = tk.Tk()
    root.withdraw()  # Ascundem fereastra principală
    file_path = filedialog.asksaveasfilename(defaultextension=".xlsx", filetypes=[('Excel Files', '*.xlsx')])
    root.destroy()  # Închidem fereastra principală
    
    if not file_path:
        return

    with pd.ExcelWriter(file_path) as writer:
        for company, data in result_data.items():
            data.to_excel(writer, sheet_name=company, index=False)

    print(f"Datele au fost salvate în: {os.path.basename(file_path)}")

def create_advanced_interface_with_drivers():
    root = tk.Tk()
    root.title('Procesare Date Transport')

    # Adding buttons and labels
    Label(root, text='Procesare Date Transport', font=("Arial", 20)).grid(row=0, columnspan=4, pady=10)

    Label(root, text='Selectează fișierul TRIP').grid(row=1, column=0, padx=10, pady=10, sticky='e')
    Button(root, text='Încarcă', command=load_trip).grid(row=1, column=1, padx=10, pady=10)

    Label(root, text='Selectează fișierul Factura 7 zile').grid(row=1, column=2, padx=10, pady=10, sticky='e')
    Button(root, text='Încarcă', command=load_invoice7).grid(row=1, column=3, padx=10, pady=10)

    Label(root, text='Selectează fișierul Factura 30 zile').grid(row=2, column=0, padx=10, pady=10, sticky='e')
    Button(root, text='Încarcă', command=load_invoice30).grid(row=2, column=1, padx=10, pady=10)

    # Show data for each company
    Label(root, text='Vizualizare Date', font=("Arial", 16)).grid(row=4, columnspan=4, pady=10)
    for idx, company in enumerate(companies, start=5):
        Button(root, text=f'Vizualizează datele pentru {company}', command=lambda c=company: process_data_for_company(c)).grid(row=idx, column=0, columnspan=2, pady=5)
        Button(root, text=f'Descarcă datele pentru {company}', command=lambda c=company: export_company_data(c)).grid(row=idx, column=2, columnspan=2, pady=5)
        Button(root, text="Exportă toate datele", command=export_all_data).grid(row=len(companies) + 5, column=0, columnspan=4, pady=5)
    root.mainloop()


# Run this function to see the GUI
create_advanced_interface_with_drivers()
