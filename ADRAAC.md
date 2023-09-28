ADRAAC: Análise de desempenho de reservatórios para aproveitamento de água de chuva


```python
#O seguinte algoritmo foi desenvolvido como produto de tese desenvolvida como requisito para obtenção do título de doutora 
#no Porgrama de Pós-graduação em Engenharia Civil e Ambiental (PPGECAM) da Universidade Federal da Paraíba (UFPB)

#Através dos coeficientes CEV (Coeficiente de Eficiência Volumétrica), COV (Coeficiente de Ociosidade Volumétrica) e 
#CTV (Coeficiente de Transbordamento Volumétrico) é possível realizar a análise de desempenho de métodos de 
#dimensionamento de reservatórios para armazenamento de água de chuva para o território brasileiro, 
#em diferentes de área de captação e demanda.

#CEV: permite analisar o volume de água aproveitável médio anual por m³ de reservatório dimensionado
#COV: permite analisar o volume ocioso médio anual por m³ de reservatório dimensionado
#CTV: permite analisar o volume transbordado médio anual por m³ de reservatório dimensionado

#A análise comparativa dos diferentes métodos de dimensionamento considerados no desenvolvimento do algoritmo permite a 
# seleção do método associado aos valores ótimos dos coeficientes supracitados.

#Mais informações para uso do algorimto estão disponíveis no link: https://github.com/Cinthya-Santos/Metodologia_reservatorios_Agua_de_Chuva

#Desenvolvido por: Cinthya Santos (cinthya.santos@ifpb.edu.br)
```


```python
#Importação de pacotes
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
root.title('Dimensionamento de reservatórios para armazenamento de água de chuva')
```




    ''




```python
# Definir as dimensões da janela
root.geometry('875x600')
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
label2 = tk.Label(info_frame, text='Critérios de dimensionamento:', anchor='w')
label2.pack(fill='x', padx=10, pady=2)

label3 = tk.Label(info_frame, text='CEV = menor tempo de retorno;', anchor='w')
label3.pack(fill='x', padx=10, pady=2)

label4 = tk.Label(info_frame, text='COV = maximização do volume de reservatório utilizado;', anchor='w')
label4.pack(fill='x', padx=10, pady=2)

label5 = tk.Label(info_frame, text='CTV = maximização do volume de água aproveitado.', anchor='w')
label5.pack(fill='x', padx=10, pady=2)
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

Funções utilizadas


```python
def seleçao_coeficiente():
    # Obtém o coeficiente selecionado pelo usuário
    coeficiente = coeficiente_selecionado.get()
    #mostra o coeficiente selecionado
    rotulo_coeficiente.config(text=f"O método de dimensionamento será baseado no {coeficiente}") 
```


```python
#Função - Reservatório Método de Rippl diário
def calcula_reservatorioRD(ponto_selecionado, area, demanda, ponto_selecionado_valores):
    # Dimensionamento dos reservatórios e cálculo dos volumes de água aproveitáveis num ciclo anual médio
    # Rippl com dados diários

    # Verificação da condição de dimensionamento
    entrada = (ponto_selecionado/1000)*0.8*area  # m³
    entrada_total = entrada.sum(dim='time')
    demanda_total = demanda*13515
    condicao = entrada_total - demanda_total  # >0 sobra água, então armazena o déficit/<0 falta água, então armazena o maior saldo.

    # Balanço Hídrico Volume do reservatório
    if condicao < 0:
        acumulo = 0  
        vresRD = 0
        # Para Demanda > Entrada
        for i in pd.Series(np.arange(0, 13515, 1)):
            oferta = ponto_selecionado_valores[i]
            acumulo = (acumulo + oferta * area * 0.0008 - demanda)
            acumulo = max(acumulo, 0)
            vresRD = max(acumulo, vresRD)
    else:
        # Para Entrada > Demanda
        acumulo = 0  
        vresRD = 0
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

    return volume_aprov_anualRD, vresRD
```


```python
#Função - Reservatório Método de Rippl mensal
def calcula_reservatorioRM(ponto_selecionado, demanda, area,ponto_selecionado_valores):
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
    oferta = precipitation_monthly_valores
    for i in pd.Series(np.arange(0, 12, 1)):
        if condicao < 0:
            acumulo += oferta[i]*area*0.0008 - demanda_valores[i]
        else:
            acumulo += demanda_valores[i] - oferta[i]*area*0.0008

        acumulo = max(0, acumulo)
        vresRM = max(vresRM, acumulo)

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
    
    return volume_aprov_anualRM, vresRM 
```


```python
#Função - Reservatório Método Azevedo Neto 210mm
def calcula_reservatorioAzN2(ponto_selecionado, area, demanda, ponto_selecionado_valores):
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
        
        volume_transb = max(volume_saldo - volume_livre, 0)
        volume_transb_total += volume_transb

    volume_aprov_anualAz2N = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)
    
    return volume_aprov_anualAz2N, vres_Az2N
```


```python
#Função - Reservatório Método Azevedo Neto var
def calcula_reservatorioVar(ponto_selecionado, area, demanda, ponto_selecionado_valores):
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
        
        volume_transb = max(volume_saldo - volume_livre, 0)
        volume_transb_total += volume_transb

    volume_aprov_anualAzN = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)
    
    return volume_aprov_anualAzN, vres_AzN
```


```python
#Função - #Função - Reservatório prático Inglês
def calcula_reservatorioIng(ponto_selecionado, area, demanda, ponto_selecionado_valores):
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
        
        volume_transb = max(volume_saldo - volume_livre, 0)
        volume_transb_total += volume_transb

    volume_aprov_anualIng = float(((ponto_selecionado.sum(dim='time') * area * 0.8 / 1000) - volume_transb_total) / 37)
    
    return volume_aprov_anualIng, vres_Ing
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
    path = 'C:/Caminho/para/os/arquivos/'
    arquivo = xr.open_mfdataset(path+'prec_daily_UT_Brazil_v2.2*.nc', combine='by_coords')
    
    # Obtém o coeficiente selecionado pelo usuário
    coeficiente = coeficiente_selecionado.get()
    
    #Verifica se o ponto pertence ao território brasileiro
    if lat >= arquivo.coords['latitude'].min() and lat <= arquivo.coords['latitude'].max() and long >= arquivo.coords['longitude'].min() and long <= arquivo.coords['longitude'].max():
        ponto_selecionado = arquivo.sel(latitude=lat,longitude=long, method='nearest')['prec'] #seleção 
        #Transformando em lista
        ponto_selecionado_valores = ponto_selecionado.values.tolist()
        
        #Leitura das variáveis
        area = float(entrada_area.get())
        demanda = float(entrada_demanda.get())/1000
        entrada_media = sum(ponto_selecionado_valores)*area*0.0008/37
        
        #Dimensionamento dos reservatórios e cálculo dos volumes de água aproveitáveis num ciclo anual médio
        funcoes_calculo = {
        'RD': calcula_reservatorioRD,
        'RM': calcula_reservatorioRM,
        'AzN2': calcula_reservatorioAzN2,
        'AzN': calcula_reservatorioVar,
        'Ing': calcula_reservatorioIng
    }
        
        # Descrições dos métodos
        metodos_desc = {
        'RD': 'Rippl com dados em base diária',
        'RM': 'Rippl com dados em base mensal',
        'AzN2': 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 210mm',
        'AzN': 'Azevedo Neto, para precipitação média mensal do mês seco inferior a 80% da precipitação média do mês',
        'Ing': 'prático Inglês'
    }
        
        # Dicionário para armazenar resultados
        resultados = {}
        
        #Cálculo coeficientes
        
        # Calcular coeficientes
        if coeficiente == 'CEV':
            for key, function in funcoes_calculo.items():
                volume_aprov_anual, vres = function(ponto_selecionado, area, demanda, ponto_selecionado_valores)

                CEV = 0 if vres == 0 else round(volume_aprov_anual / vres, 3)

                resultados[key] = {
                    'volume aproveitável anual': volume_aprov_anual,
                    'vres': vres,
                    'CEV': CEV,
                    'descricao': metodos_desc[key]
                }

            # Encontrar o CEV ótimo
            metodo_otimo = max(resultados, key=lambda k: resultados[k]['CEV'])

            CEV_otimo = resultados[metodo_otimo]['descricao']
            metodo = metodos_desc[metodo_otimo]
            vres = resultados[metodo_otimo]['vres']
            vaprov = resultados[metodo_otimo]['volume aproveitável anual']
            
            rotulo_coeficiente_otimo.config(text=f"O CEV máximo é de  {str(CEV)} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de  {str(round(vres,3))} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de  {str(round(vaprov,3))} m³")          
           
        elif coeficiente == 'COV':    
            for key, function in funcoes_calculo.items():
                volume_aprov_anual, vres = function(ponto_selecionado, area, demanda, ponto_selecionado_valores)
        
                COV = 0 if vres == 0 else round((vres * 365.27027 - volume_aprov_anual) / vres, 3)
       
                resultados[key] = {
                    'volume aproveitável anual': volume_aprov_anual,
                    'vres': vres,
                    'COV': COV,
                    'descricao': metodos_desc[key]
                }

            # Filtrar COVs maiores que zero e encontrar o método com COV mínimo
            COVs_maior_que_zero = {key: value for key, value in resultados.items() if value['COV'] > 0}
            metodo_otimo_key = min(COVs_maior_que_zero, key=lambda k: COVs_maior_que_zero[k]['COV'])

            COV_otimo = resultados[metodo_otimo_key]['COV']
            metodo = metodos_desc[metodo_otimo_key]
            vres = resultados[metodo_otimo_key]['vres']
            vaprov = resultados[metodo_otimo_key]['volume aproveitável anual']
            
            rotulo_coeficiente_otimo.config(text=f"O COV mínimo é de {COV_otimo} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de {round(vres, 3)} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de {round(vaprov, 3)} m³")  
            
        else:
            for key, function in funcoes_calculo.items():
                volume_aprov_anual, vres = function(ponto_selecionado, area, demanda, ponto_selecionado_valores)
        
                CTV = 0 if vres == 0 else round((entrada_media-volume_aprov_anual)/ vres, 3)
       
                resultados[key] = {
                    'volume aproveitável anual': volume_aprov_anual,
                    'vres': vres,
                    'CTV': CTV,
                    'descricao': metodos_desc[key]
                }

            # Filtrar COVs maiores que zero e encontrar o método com COV mínimo
            CTVs_maior_que_zero = {key: value for key, value in resultados.items() if value['CTV'] > 0}
            metodo_otimo_key = min(CTVs_maior_que_zero, key=lambda k: CTVs_maior_que_zero[k]['CTV'])

            CTV_otimo = resultados[metodo_otimo_key]['CTV']
            metodo = metodos_desc[metodo_otimo_key]
            vres = resultados[metodo_otimo_key]['vres']
            vaprov = resultados[metodo_otimo_key]['volume aproveitável anual']
            
            rotulo_coeficiente_otimo.config(text=f"O CTV mínimo é de {CTV_otimo} m³/m³ e foi obtido pelo método {metodo}")
            rotulo_vres.config(text=f"O volume do reservatório é de {round(vres, 3)} m³")
            rotulo_vaprov.config(text=f"O volume de água aproveitável médio num ciclo anual é de {round(vaprov, 3)} m³") 

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
