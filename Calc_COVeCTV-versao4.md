Cálculo dos coeficientes COV e CTV


```python
#O seguinte algoritmo foi desenvolvido como produto de tese desenvolvida como requisito para obtenção do título de doutora 
#no Porgrama de Pós-graduação em Engenharia Civil e Ambiental (PPGECAM) da Universidade Federal da Paraíba (UFPB)

#Através dos coeficientes COV (Coeficiente de Ociosidade Volumétrica) e CTV (Coeficiente de Transbordamento Volumétrico) 
#é possível realizar a análise de desempenho de métodos de dimensionamento de reservatórios para armazenamento de água de 
#chuva para o território brasileiro, em diferentes de área de captação e demanda.

#COV: permite analisar o volume ocioso médio anual por m³ de reservatório dimensionado
#CTV: permite analisar o volume transbordado médio anual por m³ de reservatório dimensionado

#A análise comparativa dos diferentes métodos de dimensionamento considerados no desenvolvimento do algoritmo permite a 
# seleção do método associado aos valores ótimos dos coeficientes supracitados.

#Mais informações para uso do algorimto estão disponíveis no link: https://github.com/Cinthya-Santos/Metodologia_reservatorios_Agua_de_Chuva

#Desenvolvido por: Cinthya Santos (cinthya.santos@ifpb.edu.br)
```


```python
import tkinter as tk
from tkinter import filedialog
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
root.title('Cálculo COV e CTV')
```




    ''




```python
# Definir as dimensões da janela
root.geometry('975x650')
```




    ''




```python
# cria um rótulo com irfomações
label = tk.Label(root, text='Desenvolvido por Cinthya Santos - PPGECAM/UFPB')
label.pack()
```


```python
# Cria um quadro para conter as informações
info_frame = tk.Frame(root, bd=2, relief="groove")
info_frame.pack(padx=10, pady=10, anchor='w')
```


```python
# Cria um rótulo com informações
label2 = tk.Label(info_frame, text='RD - Rippl com dados diários', anchor='w')
label2.pack(fill='x', padx=10, pady=2)

label3 = tk.Label(info_frame, text='RM - Rippl com dados mensais', anchor='w')
label3.pack(fill='x', padx=10, pady=2)

label4 = tk.Label(info_frame, text='AzN - método Azevedo Neto, para precipitação média mensal do mês seco inferior a 80% da precipitação média do mês', anchor='w')
label4.pack(fill='x', padx=10, pady=2)

label5 = tk.Label(info_frame, text='Az2N - método Azevedo Neto, para precipitação média mensal do mês seco inferior a 210mm', anchor='w')
label5.pack(fill='x', padx=10, pady=2)

label6 = tk.Label(info_frame, text='Ing - método Inglês', anchor='w')
label6.pack(fill='x', padx=10, pady=2)
```


```python
# Define as opções disponíveis
opcoes_metodos = ['RD', 'RM', 'AzN', 'Az2N', 'Ing']
```


```python
# Cria um objeto StringVar para armazenar a opção selecionada
metodo_selecionado = tk.StringVar(value=opcoes_metodos[0])
```


```python
# Cria um label e um menu dropdown para o usuário selecionar o coeficiente
label_metodo = tk.Label(root, text='Escolha o método para obtenção dos coeficientes:')
label_metodo.pack()
menu_metodo = tk.OptionMenu(root, metodo_selecionado, *opcoes_metodos)
menu_metodo.pack()
```


```python
def seleçao_metodo():
    # Obtém o coeficiente selecionado pelo usuário
    metodo = metodo_selecionado.get()
```


```python
# Cria um botão para seleção do coeficinte
bt1 = tk.Button(root, text='Seleção de método de dimensionamento', command=lambda: seleçao_metodo())
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
def Calculo_coef():
    
    # Seleciona o ponto de análise
    lat = float(entrada_latitude.get())
    long = float(entrada_longitude.get())
    
     # Abre o arquivo NetCDF
    path = 'C:/Caminho/para/os/arquivos/'
    arquivo = xr.open_mfdataset(path+'prec_daily_UT_Brazil_v2.2*.nc', combine='by_coords')
    
    # Obtém o coeficiente selecionado pelo usuário
    metodo = metodo_selecionado.get()
    
    #Verifica se o ponto pertence ao território brasileiro
    if lat >= arquivo.coords['latitude'].min() and lat <= arquivo.coords['latitude'].max() and long >= arquivo.coords['longitude'].min() and long <= arquivo.coords['longitude'].max():
        ponto_selecionado = arquivo.sel(latitude=lat,longitude=long, method='nearest')['prec'] #seleção 
        #Transformando em lista
        ponto_selecionado_valores = ponto_selecionado.values.tolist()
        
        #Leitura das variáveis
        area = float(entrada_area.get())
        demanda = float(entrada_demanda.get())/1000
        
        # Dimensionamento dos reservatórios e cálculo dos volumes de água aproveitáveis num ciclo anual médio
        if metodo == 'RD': # Rippl com dados diários
            # Verificação da condição de dimensionamento
            entrada = (ponto_selecionado/1000)*0.8*area  # m³
            entrada_total = entrada.sum(dim='time')
            demanda_total = demanda*13515
            condicao = entrada_total - demanda_total  # >0 sobra água, então armazena o déficit/<0 falta água, então armazena o maior saldo.
            
            # Volume do reservatório
            acumulo = 0  
            vresRD = 0
            if condicao < 0:
#                 acumulo = 0  
#                 vresRD = 0
                # Para Demanda > Entrada
                for i in pd.Series(np.arange(0, 13515, 1)):
                    oferta = ponto_selecionado_valores[i]
                    acumulo = (acumulo + oferta * area * 0.0008 - demanda)
                    acumulo = max(acumulo, 0)
                    vresRD = max(acumulo, vresRD)
            else:
                # Para Entrada > Demanda
#                 acumulo = 0  
#                 vresRD = 0
                for i in pd.Series(np.arange(0, 13515, 1)):
                    oferta = ponto_selecionado_valores[i]
                    acumulo = (acumulo + demanda - oferta * area * 0.0008)
                    acumulo = max(acumulo, 0)
                    vresRD = max(acumulo, vresRD)
            
            # Balanço hídrico cálculo do volume de água aproveitável
            volume_reservado = 0
            volume_transb_total = 0
            for i in pd.Series(np.arange(0, 13515, 1)):
                volume_saldo = (ponto_selecionado_valores[i] * area * 0.8 / 1000) - demanda
                volume_livre = vresRD - volume_reservado
                volume_reservado += volume_saldo
                volume_reservado = min(max(volume_reservado, 0), vresRD)
                volume_transb = volume_saldo - volume_livre
                volume_transb = max(volume_transb, 0)
                volume_transb_total += volume_transb

            volume_aprov_anualRD = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)

            #Cálculo dos coeficientes
            if vresRD > 0:
                COV_RD = round((vresRD*365.27027-volume_aprov_anualRD)/vresRD,3)
                entrada = sum(ponto_selecionado_valores)*area*0.0008
                CTV_RD = round(((entrada/37)-volume_aprov_anualRD)/vresRD,3)
                if CTV_RD == 0:
                    rotulo_NULO.config(text=f"O transbordamento obtido pelo reservatório é igual a zero, logo CTV= 0")  
                else:
                    rotulo_CTV.config(text=f"O CTV para o reservatórios dimensionado é de  {str(round(CTV_RD,3))} m³/m³")
        
                rotulo_COV.config(text=f"O COV para o reservatórios dimensionado é de  {str(round(COV_RD,3))} m³/m³")
                rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vresRD,3))} m³")
                rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(volume_aprov_anualRD,3))} m³")
            else:
                rotulo_vres.config(text=f"O método não permite o dimensionamento do reservatório, logo COV e CTV não podem ser cálculados")

        elif metodo == 'RM': #Rippl com dados mensais
            # Determinação da precipitação média mensal em m³ para cada ano
            demanda_dia_a_dia = (demanda + ponto_selecionado) - ponto_selecionado 
            precipitation_monthly = (ponto_selecionado.groupby('time.month').sum('time'))/37
            demanda_mensal = (demanda_dia_a_dia.groupby('time.month').sum('time'))/37

            # Verificação da condição de dimensionamento
            entrada_mensal = ((precipitation_monthly/1000)*0.8*area)
            entrada_total = entrada_mensal.sum(dim='month')
            demanda_total = demanda_mensal.sum(dim='month')
            condicao = entrada_total - demanda_total

            # Transformando em lista
            precipitation_monthly_valores = precipitation_monthly.values.tolist()
            demanda_valores = demanda_mensal.values.tolist()

            # Dimensionamento do reservatório
            acumulo = 0
            vresRM = 0 
            if condicao <0: #Demanda > Entrada
#                 acumulo = 0
#                 vresRM = 0 
                for i in pd.Series(np.arange(0, 12, 1)):
                    acumulo = acumulo + (precipitation_monthly_valores[i] * area * 0.0008) - demanda_valores[i]
                    acumulo = max(acumulo,0)
                    vresRM = max (acumulo, vresRM)
            else: #Entrada > Demanda
#                 acumulo = 0
#                 vresRM = 0 
                for i in pd.Series(np.arange(0, 12, 1)):
                    acumulo = acumulo + demanda_valores[i] - (precipitation_monthly_valores[i] * area * 0.0008) 
                    acumulo = max(acumulo,0)
                    vresRM = max (acumulo, vresRM)

            # Balanço hídrico cálculo do volume de água aproveitável
            volume_reservado = 0
            volume_transb_total = 0
            for i in pd.Series(np.arange(0, 13515, 1)):
                volume_saldo = (ponto_selecionado_valores[i]*area*0.8/1000) - demanda
                volume_livre = vresRM - volume_reservado
                volume_reservado += volume_saldo
                volume_reservado = min(max(0, volume_reservado), vresRM)
                volume_transb = volume_saldo - volume_livre
                volume_transb = max(0, volume_transb)
                volume_transb_total += volume_transb             

            volume_aprov_anualRM = float(((ponto_selecionado.sum(dim='time')*area*0.8/1000) - volume_transb_total)/37)
    
             #Cálculo dos coeficientes
            if vresRM > 0:
                COV_RM = round((vresRM*365.27027-volume_aprov_anualRM)/vresRM,3)
                entrada = sum(ponto_selecionado_valores)*area*0.0008
                CTV_RM = round(((entrada/37)-volume_aprov_anualRM)/vresRM,3)
                if CTV_RM == 0:
                    rotulo_NULO.config(text=f"O transbordamento obtido pelo reservatório é igual a zero, logo CTV= 0")  
                else:
                    rotulo_CTV.config(text=f"O CTV para o reservatórios dimensionado é de  {str(round(CTV_RM,3))} m³/m³")
        
                rotulo_COV.config(text=f"O COV para o reservatórios dimensionado é de  {str(round(COV_RM,3))} m³/m³")
                rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vresRM,3))} m³")
                rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(volume_aprov_anualRM,3))} m³")
            else:
                rotulo_vres.config(text=f"O método não permite o dimensionamento do reservatório, logo COV e CTV não podem ser cálculados")
            
        elif metodo == 'Az2N': #Azevedo Neto 210mm   
            # Agrupamento das precipitações
            precipitation_anual = ponto_selecionado.groupby('time.year').sum('time')  # agrupamento da precipitação anual em mm
            precipitation_media_anual = (precipitation_anual.sum(dim='year'))/37   # precipitação média anual em mm
            precipitation_media_mensal = ponto_selecionado.groupby('time.month').sum('time')/37  # precipitação média mensal em mm

            # Contabilização dos meses secos
            months = [1,2,3,4,5,6,7,8,9,10,11,12]
            for i in months:
                meses_secos = precipitation_media_mensal.sel(month=i).where(precipitation_media_mensal<=210,0)
            meses_secos = meses_secos.where(meses_secos == 0, 1)  # 0 chuvoso e 1 seco
            total_meses_secos = meses_secos.sum(dim='month')

            # Cálculo do volume do reservatório
            vres_Az2N = float((0.042 * precipitation_media_anual * area * total_meses_secos)/1000) # cálculo do volume do reservatório em [m³]

            # Balanço hídrico cálculo do volume de água aproveitável
            volume_reservado = 0
            volume_transb_total = 0
            for i in pd.Series(np.arange(0,13515,1)):
                volume_saldo = (ponto_selecionado_valores[i] * area * 0.8 / 1000) - demanda
                volume_livre = vres_Az2N - volume_reservado
                volume_reservado += volume_saldo
                volume_reservado = min(max(volume_reservado, 0), vres_Az2N)
                volume_transb = volume_saldo - volume_livre
                volume_transb = max(0, volume_transb)
                volume_transb_total += volume_transb

            volume_aprov_anualAz2N = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)

            #Cálculo dos coeficientes
            if vres_Az2N > 0:
                COV_Az2N = round((vres_Az2N*365.27027-volume_aprov_anualAz2N)/vres_Az2N,3)
                entrada = sum(ponto_selecionado_valores)*area*0.0008
                CTV_Az2N = round(((entrada/37)-volume_aprov_anualAz2N)/vres_Az2N,3)
                if CTV_Az2N == 0:
                    rotulo_NULO.config(text=f"O transbordamento obtido pelo reservatório é igual a zero, logo CTV= 0")  
                else:
                    rotulo_CTV.config(text=f"O CTV para o reservatórios dimensionado é de  {str(round(CTV_Az2N,3))} m³/m³")
        
                rotulo_COV.config(text=f"O COV para o reservatórios dimensionado é de  {str(round(COV_Az2N,3))} m³/m³")
                rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres_Az2N,3))} m³")
                rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(volume_aprov_anualAz2N,3))} m³")
            else:
                rotulo_vres.config(text=f"O método não permite o dimensionamento do reservatório, logo COV e CTV não podem ser cálculados")
            
        elif metodo == 'AzN': #Azevedo Neto var   
            # Agrupamento das precipitações
            precipitation_anual = ponto_selecionado.groupby('time.year').sum('time')  # agrupamento da precipitação anual em mm
            precipitation_media_anual = (precipitation_anual.sum(dim='year'))/37   # precipitação média anual em mm
            precipitation_media_mensal = ponto_selecionado.groupby('time.month').sum('time')/37  # precipitação média mensal em mm
            precipitation_media_do_mes= (precipitation_media_anual)/12   #precipitação média de cada mês no ano em mm

            # Contabilização dos meses secos
            months = [1,2,3,4,5,6,7,8,9,10,11,12]
            for i in months:
                meses_secos = precipitation_media_mensal.sel(month=i).where(precipitation_media_mensal<0.8*precipitation_media_do_mes,0)
            meses_secos = meses_secos.where(meses_secos == 0, 1)  # 0 chuvoso e 1 seco
            total_meses_secos = meses_secos.sum(dim='month')

            # Cálculo do volume do reservatório
            vres_AzN = float((0.042 * precipitation_media_anual * area * total_meses_secos)/1000) # cálculo do volume do reservatório em [m³]

            # Balanço hídrico cálculo do volume de água aproveitável
            volume_reservado = 0
            volume_transb_total = 0
            for i in pd.Series(np.arange(0,13515,1)):
                volume_saldo = (ponto_selecionado_valores[i] * area * 0.8 / 1000) - demanda
                volume_livre = vres_AzN - volume_reservado
                volume_reservado += volume_saldo
                volume_reservado = min(max(volume_reservado, 0), vres_AzN)
                volume_transb = volume_saldo - volume_livre
                volume_transb = max(0, volume_transb)
                volume_transb_total += volume_transb

            volume_aprov_anualAzN = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)       
            
            #Cálculo dos coeficientes
            if vres_AzN > 0:
                COV_AzN = round((vres_AzN*365.27027-volume_aprov_anualAzN)/vres_AzN,3)
                entrada = sum(ponto_selecionado_valores)*area*0.0008
                CTV_AzN = round(((entrada/37)-volume_aprov_anualAzN)/vres_AzN,3)
                if CTV_AzN == 0:
                    rotulo_NULO.config(text=f"O transbordamento obtido pelo reservatório é igual a zero, logo CTV= 0")  
                else:
                    rotulo_CTV.config(text=f"O CTV para o reservatórios dimensionado é de  {str(round(CTV_AzN,3))} m³/m³")
        
                rotulo_COV.config(text=f"O COV para o reservatórios dimensionado é de  {str(round(COV_AzN,3))} m³/m³")
                rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres_AzN,3))} m³")
                rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(volume_aprov_anualAzN,3))} m³")
            else:
                rotulo_vres.config(text=f"O método não permite o dimensionamento do reservatório, logo COV e CTV não podem ser cálculados")
                                 
        else: #Prático Inglês
            # Agrupamento das precipitações
            precipitation_group = ponto_selecionado.groupby('time.year').sum('time')  # agrupamento da precipitação anual em mm
            precipitation_group = (precipitation_group.sum(dim='year'))/37   # precipitação média anual em mm

            # Cálculo do volume do reservatório
            vres_Ing = float((0.05 * area * precipitation_group) / 1000) 

            # Balanço hídrico cálculo do volume de água aproveitável
            volume_reservado = 0
            volume_transb_total = 0
            for i in pd.Series(np.arange(0, 13515, 1)):
                volume_saldo = (ponto_selecionado_valores[i] * area * 0.8 / 1000) - demanda
                volume_livre = vres_Ing - volume_reservado
                volume_reservado += volume_saldo
                volume_reservado = min(max(volume_reservado, 0), vres_Ing)
                volume_transb = volume_saldo - volume_livre
                volume_transb = max(0, volume_transb)
                volume_transb_total += volume_transb

            volume_aprov_anualIng = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)
            
            #Cálculo dos coeficientes
            if vres_Ing > 0:
                COV_Ing = round((vres_Ing*365.27027-volume_aprov_anualIng)/vres_Ing,3)
                entrada = sum(ponto_selecionado_valores)*area*0.0008
                CTV_Ing = round(((entrada/37)-volume_aprov_anualIng)/vres_Ing,3)
                if CTV_Ing == 0:
                    rotulo_NULO.config(text=f"O transbordamento obtido pelo reservatório é igual a zero, logo CTV= 0")  
                else:
                    rotulo_CTV.config(text=f"O CTV para o reservatórios dimensionado é de  {str(round(CTV_Ing,3))} m³/m³")
        
                rotulo_COV.config(text=f"O COV para o reservatórios dimensionado é de  {str(round(COV_Ing,3))} m³/m³")
                rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres_Ing,3))} m³")
                rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(volume_aprov_anualIng,3))} m³")
            else:
                rotulo_vres.config(text=f"O método não permite o dimensionamento do reservatório, logo COV e CTV não podem ser cálculados")
                                 
    else:
            rotulo_ERRO.config(text=f"As coordenadas fornecidas não correspondem a um ponto válido no território brasileiro") 
# Fecha o arquivo NetCDF
    arquivo.close()  
```


```python
# Cria um botão para seleção do ponto
bt2 = tk.Button(root, text='Cálculo', command=lambda: Calculo_coef())
bt2.pack()
```


```python
# cria os rótulos
rotulo_COV = tk.Label(root)
rotulo_COV.pack()
rotulo_CTV = tk.Label(root)
rotulo_CTV.pack()
rotulo_vres = tk.Label(root)
rotulo_vres.pack()
rotulo_vaprov = tk.Label(root)
rotulo_vaprov.pack()
rotulo_ERRO = tk.Label(root)
rotulo_ERRO.pack()
rotulo_metodo = tk.Label(root)
rotulo_metodo.pack()
```


```python
def limpar_resposta():
    rotulo_COV.config(text='')
    rotulo_CTV.config(text='')
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
