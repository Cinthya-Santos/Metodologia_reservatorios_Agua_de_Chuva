Metodologia de dimensionamento de reservatório em 26/04/2023


```python
import tkinter as tk
from tkinter import filedialog
# import os
import netCDF4 as nc
import xarray as xr
import numpy as np
from tkinter import messagebox
import pandas as pd
import sys
```


```python
# Cria a janela principal
root = tk.Tk()
root.title('Dimensionamento de reservatórios para armazenamento de água de chuva')
```


```python
# Definir as dimensões da janela
root.geometry('750x450')
```


```python
# cria um rótulo com irfomações
label = tk.Label(root, text='Desenvolvido por Cinthya Santos - PPGECAM/UFPB')
label.pack()
```


```python
# Define as opções disponíveis
opcoes_coeficiente = ['CEV', 'COV', 'CTV']
```


```python
# Cria um objeto StringVar para armazenar a opção selecionada
coeficiente_selecionado = tk.StringVar(value=opcoes_coeficiente[0])
```


```python
# Cria um label e um menu dropdown para o usuário selecionar o coeficiente
label_coeficiente = tk.Label(root, text='Escolha o coeficiente no qual será baseado o dimensionamento:')
label_coeficiente.pack()
menu_coeficiente = tk.OptionMenu(root, coeficiente_selecionado, *opcoes_coeficiente)
menu_coeficiente.pack()
```


```python
def seleçao_coeficiente():
    # Obtém o coeficiente selecionado pelo usuário
    coeficiente = coeficiente_selecionado.get()
    #mostra o coeficiente selecionado
    rotulo_coeficiente.config(text=f"O método de dimensionamento será baseado no {coeficiente}") 
```


```python
# cria um rótulo para o coeficiente selecionado
rotulo_coeficiente = tk.Label(root)
rotulo_coeficiente.pack()
```


```python
# Cria um botão para seleção do coeficinte
bt1 = tk.Button(root, text='Seleção de coeficiente', command=lambda: seleçao_coeficiente())
bt1.pack()
```

Inserção de variáveis


```python
# Cria quatro labels e entradas para o usuário inserir os dados
label_latitude = tk.Label(root, text='Insira a latitude (Ex. -7.14):')
label_latitude.pack()
entrada_latitude = tk.Entry(root)
entrada_latitude.pack()

label_longitude = tk.Label(root, text='Insira a longitude (Ex. -35.4):')
label_longitude.pack()
entrada_longitude = tk.Entry(root)
entrada_longitude.pack()

label_area = tk.Label(root, text='Insira a área de captação em m²')
label_area.pack()
entrada_area = tk.Entry(root)
entrada_area.pack()

label_demanda = tk.Label(root, text='Insira a demanda de água em L/dia:')
label_demanda.pack()
entrada_demanda = tk.Entry(root)
entrada_demanda.pack()
```


```python
#Função para dimensionamento dos reservatórios no ponto e variáveis desejadas
def Calculo_reserv():
    
    # Seleciona o ponto de análise
    lat = float(entrada_latitude.get())
    long = float(entrada_longitude.get())

     # Abre o arquivo NetCDF
    #O caminho do arquivo deve ser atualizado para o caminho da pasta onde os arquivos disponibilizados estão armazenados
    #ex. C:/Usuário/arquivos_coeficientes/CEV_50_100.nc
    path = 'C:/Caminho/para/os/dados/'
    arquivo = xr.open_mfdataset(path+'prec_daily_UT_Brazil_v2.2*.nc', combine='by_coords')
    # Obtém o coeficiente selecionado pelo usuário
    coeficiente = coeficiente_selecionado.get()
    #Verifica se o ponto pertence ao território brasileiro
    if lat >= arquivo.coords['latitude'].min() and lat <= arquivo.coords['latitude'].max() and long >= arquivo.coords['longitude'].min() and long <= arquivo.coords['longitude'].max():
        ponto_selecionado = arquivo.sel(latitude=lat,longitude=long, method='nearest')['prec'] #seleção 
    
        #Transformando em lista
        ponto_escolhido_valores = ponto_selecionado.values.tolist()
        
        #Leitura das variáveis
        area = float(entrada_area.get())
        demanda = float(entrada_demanda.get())/1000
        
        #Dimensionamento dos reservatórios e cálculo dos volumes de água aproveitáveis num ciclo anual médio
        
        #Rippl com dados diários
        #Verificação da condição de dimensionamento
        entrada = (ponto_selecionado/1000)*0.8*area  #m³
        entrada_total = entrada.sum(dim='time')
        demanda_total = demanda*13515
        condicao = entrada_total-demanda_total  #>0 sobra água, então armazena o déficit/<0 falta água, então armazena o maior saldo.
         
        #Balanço Hídrico Volume do reservatório
        
        if condicao<0:
            acumulo=0  
            vresRD=0
        #Para Demanda>Entrada
            for i in pd.Series(np.arange(0,13515,1)):
                oferta = ponto_escolhido_valores[i]
                acumulo = (acumulo + oferta*area*0.0008 - demanda)
                if acumulo>0:
                    acumulo=acumulo
                else:
                    acumulo=0
                if acumulo>vresRD:
                    vresRD=acumulo
                else:
                    vresRD=vresRD
                        
        else:
        #Para Entrada>Demanda
            acumulo=0  
            vresRD=0
            for i in pd.Series(np.arange(0,13515,1)):
                oferta = ponto_escolhido_valores[i]
                acumulo = (acumulo + demanda - oferta*area*0.0008)
                if acumulo>0:
                    acumulo=acumulo
                else:
                    acumulo=0
                if acumulo>vresRD:
                    vresRD=acumulo
                else:
                    vresRD=vresRD
        #Balanço hídrico cálculo do volume de água aproveitável
        volume_reservado = 0
        volume_transb_total = 0
        for i in pd.Series(np.arange(0,13515,1)):
            volume_saldo= (ponto_escolhido_valores[i]*area*0.8/1000)- demanda
            volume_livre = vresRD - volume_reservado
            volume_reservado = volume_saldo+volume_reservado
            if volume_reservado<vresRD:
                volume_reservado=volume_reservado
            else:
                volume_reservado=vresRD
            if volume_reservado>0:
                volume_reservado=volume_reservado
            else:
                volume_reservado=0
            volume_transb = volume_saldo - volume_livre
            if volume_transb>0:
                volume_transb=volume_transb
            else:
                volume_transb=0
            volume_transb_total = volume_transb_total+ volume_transb             
                
        volume_aprov_anualRD = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000)-volume_transb_total)/37)
        
        #Rippl com dados mensais
        #Determinação da precipitação média mensal em m³ para cada ano
        demanda_dia_a_dia = (demanda + ponto_selecionado) - ponto_selecionado 
        precipitation_monthly = (ponto_selecionado.groupby('time.month').sum('time'))/37 #precipitação média mensal
        demanda_mensal = (demanda_dia_a_dia.groupby('time.month').sum('time'))/37 #demanda mensal
                
        #Verificação da condição de dimensionamento
        entrada_mensal = ((precipitation_monthly/1000)*0.8*area)
        entrada_total = entrada_mensal.sum(dim='month')
        demanda_total = demanda_mensal.sum(dim='month')
        condicao = entrada_total-demanda_total  #>0 sobra água, então armazena o déficit/<0 falta água, então armazena o maior saldo.
                
        #Transformando em lista
        precipitation_monthly_valores = precipitation_monthly.values.tolist()
        demanda_valores = demanda_mensal.values.tolist()
                
        #Dimensionamento do reservatório
        acumulo=0  
        vresRM=0
        if condicao<0:
        #Para Demanda>Entrada
            oferta = precipitation_monthly_valores
            demanda_mensal = demanda_valores
            for i in pd.Series(np.arange(0,12,1)):
                acumulo = (acumulo + oferta[i]*area*0.0008 - demanda_mensal[i])
                if acumulo>0:
                    acumulo=acumulo
                else:
                    acumulo=0
                if acumulo>vresRM:
                    vresRM=acumulo
                else:
                    vresRM=vresRM

        else:
        #Para Entrada>Demanda
            oferta = precipitation_monthly_valores
            demanda_mensal = demanda_valores
            for i in pd.Series(np.arange(0,12,1)):
                acumulo = (acumulo + demanda_mensal[i] - oferta[i]*area*0.0008)
                if acumulo>0:
                    acumulo=acumulo
                else:
                    acumulo=0
                if acumulo>vresRM:
                    vresRM=acumulo
                else:
                    vresRM=vresRM
        #Balanço hídrico cálculo do volume de água aproveitável
        volume_reservado = 0
        volume_transb_total = 0
        for i in pd.Series(np.arange(0,13515,1)):
            volume_saldo= (ponto_escolhido_valores[i]*area*0.8/1000)- demanda
            volume_livre = vresRM - volume_reservado
            volume_reservado = volume_saldo+volume_reservado
            if volume_reservado<vresRM:
                volume_reservado=volume_reservado
            else:
                volume_reservado=vresRM
            if volume_reservado>0:
                volume_reservado=volume_reservado
            else:
                volume_reservado=0
            volume_transb = volume_saldo - volume_livre
            if volume_transb>0:
                volume_transb=volume_transb
            else:
                volume_transb=0
            volume_transb_total = volume_transb_total+ volume_transb             
                
        volume_aprov_anualRM = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000)-volume_transb_total)/37)
        
        #Azevedo Neto 210mm
        
        #Agrupamento das precipitações
        precipitation_anual = ponto_selecionado.groupby('time.year').sum('time') #agrupamento da precipitação anual em mm
        precipitation_media_anual = (precipitation_anual.sum(dim='year'))/37   #precipitação média anual em mm
        precipitation_media_mensal = ponto_selecionado.groupby('time.month').sum('time')/37  #precipitação média mensal em mm
                
        #Contabilização dos meses secos
        months = [1,2,3,4,5,6,7,8,9,10,11,12]
        for i in months:
            meses_secos=precipitation_media_mensal.sel(month=i).where(precipitation_media_mensal<=210,0)  
        meses_secos=meses_secos.where(meses_secos == 0, 1)  #0 chuvoso e 1 seco
        total_meses_secos = meses_secos.sum(dim='month')
                
        #Cálculo do volume do reservatório
        vres_Az2N = float((0.042*precipitation_media_anual*area*total_meses_secos)/1000) #cálculo do volume do reservatório em [m³]
       
        #Balanço hídrico cálculo do volume de água aproveitável
        volume_reservado = 0
        volume_transb_total = 0
        for i in pd.Series(np.arange(0,13515,1)):
            volume_saldo= (ponto_escolhido_valores[i]*area*0.8/1000)- demanda
            volume_livre = vres_Az2N - volume_reservado
            volume_reservado = volume_saldo+volume_reservado
            if volume_reservado<vres_Az2N:
                volume_reservado=volume_reservado
            else:
                volume_reservado=vres_Az2N
            if volume_reservado>0:
                volume_reservado=volume_reservado
            else:
                volume_reservado=0
            volume_transb = volume_saldo - volume_livre
            if volume_transb>0:
                volume_transb=volume_transb
            else:
                volume_transb=0
            volume_transb_total = volume_transb_total+ volume_transb             
                
        volume_aprov_anualAz2N = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000)-volume_transb_total)/37)
    
        #Azevedo Neto var
        
        #Agrupamento das precipitações
        precipitation_anual = ponto_selecionado.groupby('time.year').sum('time') #agrupamento da precipitação anual em mm
        precipitation_media_anual = (precipitation_anual.sum(dim='year'))/37   #precipitação média anual em mm
        precipitation_media_mensal = ponto_selecionado.groupby('time.month').sum('time')/37  #precipitação média mensal em mm
        precipitation_media_do_mes= (precipitation_media_anual)/12   #precipitação média de cada mês no ano em mm
                
        #Contabilização dos meses secos
        months = [1,2,3,4,5,6,7,8,9,10,11,12]
        for i in months:
            meses_secos=precipitation_media_mensal.sel(month=i).where(precipitation_media_mensal<0.8*precipitation_media_do_mes,0)  
        meses_secos=meses_secos.where(meses_secos == 0, 1)  #0 chuvoso e 1 seco
        total_meses_secos = meses_secos.sum(dim='month')
                
        #Cálculo do volume do reservatório
        vres_AzN = float((0.042*precipitation_media_anual*area*total_meses_secos)/1000) #cálculo do volume do reservatório em [m³]
                
        #Balanço hídrico cálculo do volume de água aproveitável
        volume_reservado = 0
        volume_transb_total = 0
        for i in pd.Series(np.arange(0,13515,1)):
            volume_saldo= (ponto_escolhido_valores[i]*area*0.8/1000)- demanda
            volume_livre = vres_AzN - volume_reservado
            volume_reservado = volume_saldo+volume_reservado
            if volume_reservado<vres_AzN:
                volume_reservado=volume_reservado
            else:
                volume_reservado=vres_AzN
            if volume_reservado>0:
                volume_reservado=volume_reservado
            else:
                volume_reservado=0
            volume_transb = volume_saldo - volume_livre
            if volume_transb>0:
                volume_transb=volume_transb
            else:
                volume_transb=0
            volume_transb_total = volume_transb_total+ volume_transb             
                
        volume_aprov_anualAzN = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000)-volume_transb_total)/37)        
        
        #Prático Inglês
        #Agrupamento das precipitações
        precipitation_group = ponto_selecionado.groupby('time.year').sum('time') #agrupamento da precipitação anual em mm
        precipitation_group = (precipitation_group.sum(dim='year'))/37   #precipitação média anual em mm
                
        #Cálculo do volume do reservatório
        vres_Ing = float((0.05*area*precipitation_group)/1000) 
        
        #Balanço hídrico cálculo do volume de água aproveitável
        volume_reservado = 0
        volume_transb_total = 0
        for i in pd.Series(np.arange(0,13515,1)):
            volume_saldo= (ponto_escolhido_valores[i]*area*0.8/1000)- demanda
            volume_livre = vres_Ing - volume_reservado
            volume_reservado = volume_saldo+volume_reservado
            if volume_reservado<vres_Ing:
                volume_reservado=volume_reservado
            else:
                volume_reservado=vres_Ing
            if volume_reservado>0:
                volume_reservado=volume_reservado
            else:
                volume_reservado=0
            volume_transb = volume_saldo - volume_livre
            if volume_transb>0:
                volume_transb=volume_transb
            else:
                volume_transb=0
            volume_transb_total = volume_transb_total+ volume_transb             
                
        volume_aprov_anualIng = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000)-volume_transb_total)/37)
        
        #Cálculo coeficientes
        
        if coeficiente=='CEV':
            #CEV de cada método
            CEV_RD = round(volume_aprov_anualRD/vresRD,3)
            CEV_Az2N = round(volume_aprov_anualAz2N/vres_Az2N,3)
            CEV_AzN = round(volume_aprov_anualAzN/vres_AzN,3)
            CEV_Ing = round(volume_aprov_anualIng/vres_Ing,3)
            if vresRM == 0:
                CEV_RM = 0
            else:
                CEV_RM = round(volume_aprov_anualRM/vresRM,3)
            
            #CEV Ótimo
            CEV_otimo = max(CEV_RD, CEV_RM, CEV_Az2N, CEV_AzN, CEV_Ing)
            if CEV_otimo == CEV_RD:
                metodo = 'Rippl com dados em base diária'
                vres = vresRD
                vaprov = volume_aprov_anualRD
            elif CEV_otimo == CEV_RM:
                metodo = 'Rippl com dados em base mensal'
                vres = vresRM
                vaprov = volume_aprov_anualRM
            elif CEV_otimo == CEV_Az2N:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 210mm'
                vres = vres_Az2N
                vaprov = volume_aprov_anualAz2N
            elif CEV_otimo == CEV_AzN:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 80% da precipitação média do mês'
                vres = vres_AzN
                vaprov = volume_aprov_anualAzN
            else:
                metodo = 'prático Inglês'
                vres = vres_Ing
                vaprov = volume_aprov_anualIng
            
            rotulo_coeficiente_otimo.config(text=f"O CEV máximo é de  {str(CEV_otimo)} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres,3))} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(vaprov,3))} m³")
        
        elif coeficiente == 'COV':
            #COV de cada método
            COV_RD = round((vresRD*365.27027-volume_aprov_anualRD)/vresRD,3)
            COV_Az2N = round((vres_Az2N*365.27027-volume_aprov_anualAz2N)/vres_Az2N,3)
            COV_AzN = round((vres_AzN*365.27027-volume_aprov_anualAzN)/vres_AzN,3)
            COV_Ing = round((vres_Ing*365.27027-volume_aprov_anualIng)/vres_Ing,3)
            if vresRM == 0:
                COV_RM = 0
                COV_otimo = min(COV_RD, COV_Az2N, COV_AzN, COV_Ing)
            else:
                COV_RM = round((vresRM*365.27027-volume_aprov_anualRM)/vresRM,3)
                COV_otimo = min(COV_RD, COV_RM, COV_Az2N, COV_AzN, COV_Ing)
            
            if COV_otimo == COV_RD:
                metodo = 'Rippl com dados em base diária'
                vres = vresRD
                vaprov = volume_aprov_anualRD
            elif COV_otimo == COV_RM:
                metodo = 'Rippl com dados em base mensal'
                vres = vresRM
                vaprov = volume_aprov_anualRM
            elif COV_otimo == COV_Az2N:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 210mm'
                vres = vres_Az2N
                vaprov = volume_aprov_anualAz2N
            elif COV_otimo == COV_AzN:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 80% da precipitação média do mês'
                vres = vres_AzN
                vaprov = volume_aprov_anualAzN
            else:
                metodo = 'prático Inglês'
                vres = vres_Ing
                vaprov = volume_aprov_anualIng
            
            rotulo_coeficiente_otimo.config(text=f"O COV mínimo é de  {str(COV_otimo)} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres,3))} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(vaprov,3))} m³")
        else:
            #CTV de cada método
            entrada = sum(ponto_escolhido_valores)*area*0.0008
            CTV_Az2N = round(((entrada/37)-volume_aprov_anualAz2N)/vres_Az2N,3)
            CTV_AzN = round(((entrada/37)-volume_aprov_anualAzN)/vres_AzN,3)
            CTV_Ing = round(((entrada/37)-volume_aprov_anualIng)/vres_Ing,3)
            CTV_RD = round(((entrada/37)-volume_aprov_anualRD)/vresRD,3)
            CTV_RM = round(((entrada/37)-volume_aprov_anualRM)/vresRM,3)
            lista_CTV = [CTV_RD, CTV_RM, CTV_Az2N, CTV_AzN, CTV_Ing]
            valores_positivos = list(filter(lambda x: x > 0, lista_CTV))
            CTV_otimo = min(valores_positivos)
            
            if CTV_otimo == CTV_RD:
                metodo = 'Rippl com dados em base diária'
                vres = vresRD
                vaprov = volume_aprov_anualRD
            elif CTV_otimo == CTV_RM:
                metodo = 'Rippl com dados em base mensal'
                vres = vresRM
                vaprov = volume_aprov_anualRM
            elif CTV_otimo == CTV_Az2N:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 210mm'
                vres = vres_Az2N
                vaprov = volume_aprov_anualAz2N
            elif CTV_otimo == CTV_AzN:
                metodo = 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 80% da precipitação média do mês'
                vres = vres_AzN
                vaprov = volume_aprov_anualAzN
            else:
                metodo = 'prático Inglês'
                vres = vres_Ing
                vaprov = volume_aprov_anualIng
            
            rotulo_coeficiente_otimo.config(text=f"O CTV mínimo é de  {str(CTV_otimo)} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres,3))} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(vaprov,3))} m³")
    else:
            rotulo_ERRO.config(text=f"As coordenadas fornecidas não correspondem a um ponto válido no território brasileiro") 
# Fecha o arquivo NetCDF
    arquivo.close()  
```


```python
# Cria um botão para seleção do ponto
bt2 = tk.Button(root, text='Cálculo', command=lambda: Calculo_reserv())
bt2.pack()
```


```python
# cria os rótulos
rotulo_coeficiente_otimo = tk.Label(root)
rotulo_coeficiente_otimo.pack()
rotulo_vres = tk.Label(root)
rotulo_vres.pack()
rotulo_vaprov = tk.Label(root)
rotulo_vaprov.pack()
rotulo_ERRO = tk.Label(root)
rotulo_ERRO.pack()
```


```python
def limpar_resposta():
    rotulo_coeficiente.config(text='')
    rotulo_coeficiente_otimo.config(text='')
    rotulo_vres.config(text='')
    rotulo_vaprov.config(text='')
    rotulo_ERRO.config(text='')
```


```python
# Crie um botão que chama a função limpar_resposta
bt3 = tk.Button(root, text='Limpar', command=limpar_resposta)
bt3.pack()
```


```python
# Inicia a janela principal
root.mainloop()
```
