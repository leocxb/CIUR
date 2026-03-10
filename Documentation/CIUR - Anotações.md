![[Tarefas CIUR]]

## Referências
[Hadlock]("C:\Users\Usuario\Downloads\hadlock1991.pdf")

## Etapas do desenvolvimento do projeto:

### Processamento dos Dados
1. Junção das planilhas para criar o dataset bruto com **563.886** pacientes
2. Remoção das colunas não úteis ao projeto, mantendo apenas:
   [[PID, CC, CF, DBP, CCN,  (4)(2), IG no exame ]]
3. Ordenação do dataset e verificação de [[PID + IG]] duplicados para remoção de dados
   Foram removidos **13.387** pacientes
4. Após a verificação de que haviam muitas lacunas no projeto, realizei a imputação dos dados em 5% dos dados incompletos, diminuindo o viés final
5. Verificação da paridade dos pacientes, ou seja, para ele ser mantido no dataset, ele precisaria, necessariamente, aparecer pelo menos 1 (uma) vez antes de 20 semanas e depois de 20 semanas
   Redução do dataset para **201.391** pacientes
6. Separação do dataset em 2, mantendo um dataset com informações utilizáveis e outro com informações descartadas

![[Pasted image 20260305163124.png]]

### Primeiro Modelo (Treinamento de Random Forest)
1. Verificação da completude do dataset final para descobrir quantas linhas estavam 100% completas, eliminando as demais
2. 15.220 pacientes com linhas completas, então foi treinado um modelo baseado em Random Forest para que [[CC, CF, DBP]] chegassem ao [[CCN]]
3. Utilizando Imputação Preditiva, foram recuperados **126.855** pacientes no dataset com um erro médio de 4.06mm

### Arquitetura do Modelo
1. Cruzando as informações entre **Idade Gestacional** e **Peso** com a Tabela (In Utero Fetal Weight Standards at US) de Hadlock (décimo quartil), treinei um modelo para dizer se um feto possui CIUR ou não
2. Treinando o modelo, algumas informações da tabela foram escondidas para evitar vazamento de dados, forçando o modelo a aprender a diagnosticar o CIUR apenas através das medidas ósseas e não do peso. 

![[Pasted image 20260305163158.png]]



### Segundo Modelo (Treinamento do XGBoost)
1. Através de testes com diversos tipos de modelo (Regressão Linear, Random Forest, KNN e XGBoost), foi escolhido o XGBoost por possuir a melhor acurácia (**ROC AUC** de **96.9%**)
2. Treino do modelo para determinar se o paciente possui CIUR através da tabela do Hadlock e do peso dos fetos, conseguindo ensinar a predição através das informações ósseas [[CCN, CC, CF, DBP]] indicando se [[CIUR == true/false]]
3. Para calibrar o modelo foi realizado um estudo para identificar qual seria o melhor ponto de corte, ou seja, o programa verifica os dados e apenas retorna que um paciente possui CIUR se ele tiver ao menos 70% de certeza de que a informação está correta, caso contrário, ela é tratada como um falso positivo

![[Pasted image 20260305163243.png]]


![[Pasted image 20260305163301.png]]

### Futuro do projeto
1. Como os pacientes possuem no início apenas o [[CCN]], e não [[CC, CF, DBP]], foi pedido para que o processo inverso do PRIMEIRO MODELO fosse feito, tentando prever o [[CC, CF, DBP]] através do [[CCN]] fornecido no paciente
2. Criação de um programa executável onde um usuário possa introduzir as medições de um paciente e verificar, de maneira barata se o paciente possui ou irá possuir CIUR
   -> O usuário deve poder escolher se o teste está sendo feito em um feto com menos de 20 semanas ou não, fazendo com que o programa não peça todas as informações, ou seja, se for antes de 20 semanas, o programa pedirá apenas o CCN, caso contrário, ele pedirá todas as informações, podendo deixar o CCN em branco caso não tenha ocorrido a medição
3. Tentar prever em qual momento da gestação foi desenvolvido o CIUR
4. Escrever artigo sobre o projeto?

### Modelo Bidirecional (Treinamento da Random Forest e XGBoost)
- Após reunião, decidimos testar a criação de um modelo que tente prever [[CC, CF, DBP]]
através do [[CCN]]. Ao implementar, fazemos o programa verificar todo o dataset para preencher o máximo possível de dados em branco, tentando aumentar o tamanho total. Através deste programa, conseguimos resgatar **59.311** pacientes através da predição dos dados ósseos pelo [[CCN]].
- Criando um threshold, foi possível perceber melhora na decisão do programa, uma vez que o AUC aumentou para **97.55** e o ponto de corte reduziu para 50%, reduzindo a quantidade de alarmes de falso positivo no programa e aumentando a Sensibilidade para **87.42%**

![[Pasted image 20260310144838.png]]






## Dataset

```dataviewjs
// Defina aqui o caminho exato do seu arquivo dentro do Obsidian
const caminho_do_arquivo = "CIUR/Data Modeling/accepted_PIDS_SUPER_DATASET.csv";

try {
    // Comando do DataviewJS para ler o CSV
    const dados = await dv.io.csv(caminho_do_arquivo);
    
    // Pegando apenas as 100 primeiras linhas para não travar o computador
    const amostra = dados.slice(0, 100);
    
    // Montando a tabela
    dv.table(
        ["🆔 PID", "📅 IG", "📏 CCN", "🧠 CC", "🦴 CF", "📏 DBP", "⚖️ Peso", "🤖 Imputado?"],
        amostra.map(linha => [
            linha["PID"],
            linha["IG no exame"],
            linha["CCN"],
            linha["CC"],
            linha["CF"],
            linha["DBP"],
            linha[" (4) (2)"],
            linha["row_is_virtual"] == 1 ? "Sim" : "Não"
        ])
    );
} catch (erro) {
    dv.paragraph("⚠️ **Erro:** Não foi possível encontrar o arquivo. Verifique se a pasta 'Datasets' e o arquivo CSV estão com o nome exato no seu cofre do Obsidian.");
}
```



## Códigos


### Código para tratamento de dados 

``` Python

import pandas as pd
import numpy as np
import os


# 1. FUNÇÃO DE LEITURA E UNIÃO

def merge_excel_sheets_to_dataframe(directory_path):
    dataframes = []
    print("Iniciando a leitura e união das planilhas...")
    for filename in os.listdir(directory_path):
        if filename.endswith('.xlsx') or filename.endswith('.xls'):
            file_path = os.path.join(directory_path, filename)
            try:
                excel_file = pd.ExcelFile(file_path)
                for sheet_name in excel_file.sheet_names:
                    df = pd.read_excel(file_path, sheet_name=sheet_name)
                    df['Source_File'] = filename
                    df['Sheet_Name'] = sheet_name
                    dataframes.append(df)
            except Exception as e:
                print(f"Erro ao ler o arquivo {filename}: {e}")
    
    if not dataframes:
        return pd.DataFrame()
        
    merged_dataframe = pd.concat(dataframes, ignore_index=True)
    print(f"\nUnião completa! Total de {len(merged_dataframe)} registros lidos brutos.")
    return merged_dataframe


# 2. FUNÇÃO DE IMPUTAÇÃO COM COTA LIMITADA

def conditional_imputation_with_quota(df, col_ig, cols_early, cols_late, cut_off_weeks=20, quota_pct=0.025):
    df_processed = df.copy()
    df_processed['row_is_virtual'] = 0
    total_rows = len(df_processed)
    max_changes_per_group = int(total_rows * quota_pct)
    
    print(f"\n--- INICIANDO IMPUTAÇÃO (COTA MÁXIMA DE {quota_pct*100}% POR GRUPO) ---")
    
    df_processed[col_ig] = pd.to_numeric(df_processed[col_ig], errors='coerce')
    mask_early_group = df_processed[col_ig] < cut_off_weeks
    mask_late_group = df_processed[col_ig] >= cut_off_weeks

    # Grupo Early (< 20 semanas)
    missing_early_mask = mask_early_group & (df_processed[cols_early].isnull().any(axis=1))
    candidates_early_idx = df_processed[missing_early_mask].index.tolist()
    
    if len(candidates_early_idx) > max_changes_per_group:
        selected_early_idx = np.random.choice(candidates_early_idx, size=max_changes_per_group, replace=False)
    else:
        selected_early_idx = candidates_early_idx

    if len(selected_early_idx) > 0:
        for col in cols_early:
            median_val = df_processed.loc[mask_early_group, col].median()
            df_processed.loc[selected_early_idx, col] = df_processed.loc[selected_early_idx, col].fillna(median_val)
        df_processed.loc[selected_early_idx, 'row_is_virtual'] = 1

    # Grupo Late (>= 20 semanas)
    missing_late_mask = mask_late_group & (df_processed[cols_late].isnull().any(axis=1))
    candidates_late_idx = df_processed[missing_late_mask].index.tolist()
    
    if len(candidates_late_idx) > max_changes_per_group:
        selected_late_idx = np.random.choice(candidates_late_idx, size=max_changes_per_group, replace=False)
    else:
        selected_late_idx = candidates_late_idx
        
    if len(selected_late_idx) > 0:
        for col in cols_late:
            median_val = df_processed.loc[mask_late_group, col].median()
            df_processed.loc[selected_late_idx, col] = df_processed.loc[selected_late_idx, col].fillna(median_val)
        df_processed.loc[selected_late_idx, 'row_is_virtual'] = 1
            
    return df_processed


# 3. REMOVE LINHAS VAZIAS

def drop_incomplete_rows(df, col_ig, cols_early, cols_late, cut_off=20):
    print("\n--- LIMPANDO DADOS INCOMPLETOS RESTANTES ---")
    
    df_clean = df.copy()
    df_clean[col_ig] = pd.to_numeric(df_clean[col_ig], errors='coerce')
    
    df_clean = df_clean.dropna(subset=[col_ig])
    
    fail_early = (df_clean[col_ig] < cut_off) & (df_clean[cols_early].isnull().any(axis=1))
    fail_late = (df_clean[col_ig] >= cut_off) & (df_clean[cols_late].isnull().any(axis=1))
    
    rows_to_drop_mask = fail_early | fail_late
    df_final = df_clean[~rows_to_drop_mask]
    
    print(f"Linhas mantidas após limpeza: {len(df_final)} de {len(df)}")
    return df_final


# 4. NOVA FUNÇÃO: FILTRO LONGITUDINAL

def filter_longitudinal_patients(df, pid_col, ig_col, cut_off_weeks=20):
    print("\n--- FILTRO LONGITUDINAL: EXIGINDO EXAMES ANTES E DEPOIS DE 20 SEMANAS ---")
    
    df_temp = df.copy()
    df_temp[ig_col] = pd.to_numeric(df_temp[ig_col], errors='coerce')
    df_temp = df_temp.dropna(subset=[pid_col, ig_col])
    
    initial_patients = df_temp[pid_col].nunique()
    initial_rows = len(df_temp)
    
    pids_early = set(df_temp[df_temp[ig_col] < cut_off_weeks][pid_col])
    pids_late = set(df_temp[df_temp[ig_col] >= cut_off_weeks][pid_col])
    
    # Interseção: O paciente precisa existir nos dois grupos
    valid_pids = pids_early.intersection(pids_late)
    
    df_filtered = df[df[pid_col].isin(valid_pids)].copy()
    
    final_patients = df_filtered[pid_col].nunique()
    
    print(f"Pacientes únicos antes do filtro: {initial_patients}")
    print(f"Pacientes válidos (Longitudinais): {final_patients}")
    print(f"Pacientes descartados (Sem acompanhamento completo): {initial_patients - final_patients}")
    print(f"Total de linhas resultantes: {len(df_filtered)}")
    
    return df_filtered, valid_pids


# SCRIPT PRINCIPAL


# Caminho da pasta (Ajuste se necessário)
directory_path = r'C:\Users\Usuario\Jupyter\CIUR\04-03-26\Data Treatment\planilhas'

PID_COL = 'PID'
IG_COL = 'IG no exame'
EARLY_COLS = ['CCN']
LATE_COLS = ['CC', 'CF', 'DBP']
DATA_COLS = EARLY_COLS + LATE_COLS 
OTHER_COLS = [' (4) (2)', 'IG exame'] 

# 1. Ler planilhas
merged_df = merge_excel_sheets_to_dataframe(directory_path)

if not merged_df.empty:
    if PID_COL not in merged_df.columns:
        merged_df[PID_COL] = np.nan

    # 2. Imputação Controlada
    df_imputed = conditional_imputation_with_quota(
        merged_df, IG_COL, EARLY_COLS, LATE_COLS, quota_pct=0.025
    )

    # 3. Remover vazios restantes (Strict Cleaning)
    df_cleaned = drop_incomplete_rows(
        df_imputed, IG_COL, EARLY_COLS, LATE_COLS, cut_off=20
    )

    # 4. Ordenação e Deduplicação Básica (Mesmo Paciente + Mesma IG)
    sort_cols = DATA_COLS + OTHER_COLS + [IG_COL]
    existing_sort_cols = [c for c in sort_cols if c in df_cleaned.columns]
    
    df_sorted = df_cleaned.sort_values(by=existing_sort_cols, ascending=False, ignore_index=True)
    df_sorted['is_duplicate'] = df_sorted.duplicated(subset=[PID_COL, IG_COL], keep='first')
    
    accepted_df = df_sorted[~df_sorted['is_duplicate']].copy()
    unaccepted_duplicates = df_sorted[df_sorted['is_duplicate']].copy()

    # 5. Filtro Longitudinal (A grande peneira final)
    accepted_longitudinal_df, valid_pids = filter_longitudinal_patients(
        accepted_df, PID_COL, IG_COL, cut_off_weeks=20
    )

    # Identificar as linhas que foram rejeitadas pelo filtro longitudinal para adicionar ao log de lixo
    discarded_longitudinal_df = accepted_df[~accepted_df[PID_COL].isin(valid_pids)].copy()
    
    # Unir todos os descartados (duplicatas + sem acompanhamento) no arquivo de log
    final_unaccepted_df = pd.concat([unaccepted_duplicates, discarded_longitudinal_df], ignore_index=True)

    # 6. Relatório Final
    print("\n" + "="*60)
    print("RESUMO FINAL: ")
    print("="*60)
    total_rows = len(accepted_longitudinal_df)
    virtual_rows = accepted_longitudinal_df['row_is_virtual'].sum() if 'row_is_virtual' in accepted_longitudinal_df.columns else 0
    real_rows = total_rows - virtual_rows
    
    pct_real = (real_rows / total_rows * 100) if total_rows > 0 else 0
    pct_virtual = (virtual_rows / total_rows * 100) if total_rows > 0 else 0

    print(f"TOTAL DE LINHAS PARA O MODELO: {total_rows}")
    print(f"TOTAL DE PACIENTES ÚNICOS: {accepted_longitudinal_df[PID_COL].nunique()}")
    print(f"LINHAS REAIS:    {real_rows} ({pct_real:.2f}%)")
    print(f"LINHAS VIRTUAIS: {virtual_rows} ({pct_virtual:.2f}%)")
    print("="*60)

    # 7. Salvar Arquivos
    cols_to_save = [PID_COL] + existing_sort_cols + ['row_is_virtual']
    cols_to_save = list(dict.fromkeys(cols_to_save)) # Remove duplicatas na lista de colunas
    cols_to_save = [c for c in cols_to_save if c in accepted_longitudinal_df.columns]
    
    accepted_longitudinal_df[cols_to_save].to_csv('accepted_PIDS_final_para_modelo.csv', index=False)
    final_unaccepted_df.to_csv('unaccepted_PIDS_duplicates_log.csv', index=False)
    
    print(f"\nArquivos salvos com sucesso! O 'accepted_PIDS_final_para_modelo.csv' está pronto para Machine Learning.")
else:
    print("O processo foi abortado porque nenhuma planilha pôde ser lida.")

```

### Código do Modelo

```Python
import pandas as pd
import numpy as np
from sklearn.ensemble import RandomForestRegressor
from xgboost import XGBClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import precision_score, recall_score, f1_score, roc_auc_score
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
import joblib

print("=== FASE 1: CONSTRUINDO DATASET BIDIRECIONAL ===\n")
df = pd.read_csv('accepted_PIDS_final_para_modelo.csv')

# 1. Imputação Direta (Dos Ossos para o CCN)
df_treino_dir = df.dropna(subset=['CCN', 'CC', 'CF', 'DBP'])
rf_dir = RandomForestRegressor(random_state=42, n_jobs=-1)
rf_dir.fit(df_treino_dir[['CC', 'CF', 'DBP']], df_treino_dir['CCN'])

mask_dir = df['CCN'].isnull() & df[['CC', 'CF', 'DBP']].notnull().all(axis=1)
if mask_dir.sum() > 0:
    df.loc[mask_dir, 'CCN'] = rf_dir.predict(df.loc[mask_dir, ['CC', 'CF', 'DBP']])
print(f"-> CCNs resgatados com sucesso: {mask_dir.sum()}")

# 2. Imputação Reversa (Do CCN + IG para os Ossos)
df_treino_rev = df.dropna(subset=['CCN', 'CC', 'CF', 'DBP', 'IG no exame'])
rf_rev = RandomForestRegressor(random_state=42, n_jobs=-1)
rf_rev.fit(df_treino_rev[['CCN', 'IG no exame']], df_treino_rev[['CC', 'CF', 'DBP']])

mask_rev = df['CCN'].notnull() & df['IG no exame'].notnull() & df[['CC', 'CF', 'DBP']].isnull().any(axis=1)
if mask_rev.sum() > 0:
    df.loc[mask_rev, ['CC', 'CF', 'DBP']] = rf_rev.predict(df.loc[mask_rev, ['CCN', 'IG no exame']])
print(f"-> Biometrias precoces resgatadas com sucesso: {mask_rev.sum()}")

df.to_csv('accepted_PIDS_SUPER_DATASET.csv', index=False)
print("\n[+] 'accepted_PIDS_SUPER_DATASET.csv' salvo com sucesso!")


print("\n=== FASE 2: TREINANDO O NOVO MODELO ===\n")
df_tabela = pd.read_csv('Table1.csv')
df['IG_arredondada'] = df['IG no exame'].apply(np.floor)
df = df.merge(df_tabela, left_on='IG_arredondada', right_on='Menstrual Week', how='left')
df['CIUR'] = (df[' (4) (2)'] < df['10th Percentile (g)']).astype(int)

colunas_remover = [
    'PID', 'CIUR', ' (4) (2)', 'row_is_virtual', 'IG_arredondada',
    'Menstrual Week', '3rd Percentile (g)', '10th Percentile (g)',
    '50th Percentile (g)', '90th Percentile (g)', '97th Percentile (g)', 'IG exame'
]
X = df.drop(columns=colunas_remover, errors='ignore')
y = df['CIUR']

# Separar treino e teste
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

# Configurar o modelo com o peso para não perder doentes
razao = len(y_train[y_train == 0]) / len(y_train[y_train == 1])
pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler()),
    ('xgb', XGBClassifier(random_state=42, eval_metric='logloss', scale_pos_weight=razao))
])

print("Treinando o XGBoost... (Isso pode levar alguns segundos)")
pipeline.fit(X_train, y_train)

# Avaliar
probs = pipeline.predict_proba(X_test)[:, 1]
roc = roc_auc_score(y_test, probs)
print(f"NOVO ROC AUC ALCANÇADO: {roc * 100:.2f}%\n")

print("=== TABELA DE DESEMPENHO ATUALIZADA ===")
cortes = [0.50, 0.60, 0.70, 0.80, 0.90]
resultados = []
for c in cortes:
    prev = (probs >= c).astype(int)
    resultados.append({
        'Corte (Threshold)': f"{c*100}%",
        'Sensibilidade (Acha doentes)': round(recall_score(y_test, prev), 4),
        'Precisão (Alarmes corretos)': round(precision_score(y_test, prev), 4),
        'F1-Score': round(f1_score(y_test, prev), 4)
    })

df_resultados = pd.DataFrame(resultados)
print(df_resultados.to_markdown(index=False))

# Substituir o modelo antigo pelo novo
joblib.dump(pipeline, 'modelo_ciur_xgboost_final.pkl')
print("\nArquivo 'modelo_ciur_xgboost_final.pkl' foi atualizado!")

```

#### Código para tabela em porcentagem 

```Python

from sklearn.metrics import accuracy_score, confusion_matrix
import pandas as pd

print("=== TABELA CLÍNICA DEFINITIVA PARA O ARTIGO ===\n")

cortes = [0.50, 0.60, 0.70, 0.80, 0.90]
resultados_artigo = []

for c in cortes:
    prev = (probs >= c).astype(int)
    tn, fp, fn, tp = confusion_matrix(y_test, prev).ravel()
    
    especificidade = tn / (tn + fp) if (tn + fp) > 0 else 0
    vpn = tn / (tn + fn) if (tn + fn) > 0 else 0 
    
    resultados_artigo.append({
        'Corte': f"{c*100:.0f}%",
        'Acurácia Global': f"{accuracy_score(y_test, prev) * 100:.2f}%",
        'Sensibilidade': f"{recall_score(y_test, prev) * 100:.2f}%",
        'Especificidade': f"{especificidade * 100:.2f}%",
        'Precisão (VPP)': f"{precision_score(y_test, prev) * 100:.2f}%",
        'VPN': f"{vpn * 100:.2f}%",
        'F1-Score': f"{f1_score(y_test, prev) * 100:.2f}%"
    })

df_artigo = pd.DataFrame(resultados_artigo)
print(df_artigo.to_markdown(index=False))
```


### Criação do Código do Aplicativo
#### Objetivo:

Criar um aplicativo para o projeto CIUR contendo um seletor de período de gestação para que o modelo preveja se o feto terá CIUR ou se ele é CIUR positivo

## Código do Modelo Bidirecional

```python
import pandas as pd
import numpy as np
import joblib
from sklearn.ensemble import RandomForestRegressor
from xgboost import XGBClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import precision_score, recall_score, f1_score, roc_auc_score, accuracy_score, confusion_matrix
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

print("=== FASE 1: CONSTRUINDO DATASET BIDIRECIONAL ===\n")
df = pd.read_csv('accepted_PIDS_final_para_modelo.csv')

# 1. Imputação Direta (Dos Ossos + IG para o CCN)
# Adicionado 'IG no exame' nas features para melhorar a precisão da imputação
df_treino_dir = df.dropna(subset=['CCN', 'CC', 'CF', 'DBP', 'IG no exame'])
rf_dir = RandomForestRegressor(random_state=42, n_jobs=-1)
rf_dir.fit(df_treino_dir[['CC', 'CF', 'DBP', 'IG no exame']], df_treino_dir['CCN'])

mask_dir = df['CCN'].isnull() & df[['CC', 'CF', 'DBP', 'IG no exame']].notnull().all(axis=1)
if mask_dir.sum() > 0:
    df.loc[mask_dir, 'CCN'] = rf_dir.predict(df.loc[mask_dir, ['CC', 'CF', 'DBP', 'IG no exame']])
print(f"-> CCNs resgatados com sucesso: {mask_dir.sum()}")

# 2. Imputação Reversa (Do CCN + IG para os Ossos)
df_treino_rev = df.dropna(subset=['CCN', 'CC', 'CF', 'DBP', 'IG no exame'])
rf_rev = RandomForestRegressor(random_state=42, n_jobs=-1)
rf_rev.fit(df_treino_rev[['CCN', 'IG no exame']], df_treino_rev[['CC', 'CF', 'DBP']])

mask_rev = df['CCN'].notnull() & df['IG no exame'].notnull() & df[['CC', 'CF', 'DBP']].isnull().any(axis=1)
if mask_rev.sum() > 0:
    df.loc[mask_rev, ['CC', 'CF', 'DBP']] = rf_rev.predict(df.loc[mask_rev, ['CCN', 'IG no exame']])
print(f"-> Biometrias precoces resgatadas com sucesso: {mask_rev.sum()}")

df.to_csv('accepted_PIDS_SUPER_DATASET.csv', index=False)
print("\n[+] 'accepted_PIDS_SUPER_DATASET.csv' salvo com sucesso!")


print("\n=== FASE 2: TREINANDO O NOVO MODELO ===\n")
df_tabela = pd.read_csv('Table1.csv')
df['IG_arredondada'] = df['IG no exame'].apply(np.floor)
df = df.merge(df_tabela, left_on='IG_arredondada', right_on='Menstrual Week', how='left')
df['CIUR'] = (df[' (4) (2)'] < df['10th Percentile (g)']).astype(int)

# ATENÇÃO: 'IG exame' e 'IG no exame' não estão mais na lista de remoção. 
# A idade gestacional agora será usada no treinamento do XGBoost!
colunas_remover = [
    'PID', 'CIUR', ' (4) (2)', 'row_is_virtual', 'IG_arredondada',
    'Menstrual Week', '3rd Percentile (g)', '10th Percentile (g)',
    '50th Percentile (g)', '90th Percentile (g)', '97th Percentile (g)'
]
X = df.drop(columns=colunas_remover, errors='ignore')
y = df['CIUR']

# Separar treino e teste
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

# Configurar o modelo com o peso para não perder doentes
razao = len(y_train[y_train == 0]) / len(y_train[y_train == 1])
pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler()),
    ('xgb', XGBClassifier(random_state=42, eval_metric='logloss', scale_pos_weight=razao))
])

print("Treinando o XGBoost... (Isso pode levar alguns segundos)")
pipeline.fit(X_train, y_train)

# Avaliar
probs = pipeline.predict_proba(X_test)[:, 1]
roc = roc_auc_score(y_test, probs)
print(f"NOVO ROC AUC ALCANÇADO: {roc * 100:.2f}%\n")


print("=== TABELA CLÍNICA DEFINITIVA PARA O ARTIGO ===\n")
cortes = [0.50, 0.60, 0.70, 0.80, 0.90]
resultados_artigo = []

for c in cortes:
    prev = (probs >= c).astype(int)
    tn, fp, fn, tp = confusion_matrix(y_test, prev).ravel()
    
    especificidade = tn / (tn + fp) if (tn + fp) > 0 else 0
    vpn = tn / (tn + fn) if (tn + fn) > 0 else 0 
    
    resultados_artigo.append({
        'Corte': f"{c*100:.0f}%",
        'Acurácia Global': f"{accuracy_score(y_test, prev) * 100:.2f}%",
        'Sensibilidade': f"{recall_score(y_test, prev) * 100:.2f}%",
        'Especificidade': f"{especificidade * 100:.2f}%",
        'Precisão (VPP)': f"{precision_score(y_test, prev) * 100:.2f}%",
        'VPN': f"{vpn * 100:.2f}%",
        'F1-Score': f"{f1_score(y_test, prev) * 100:.2f}%"
    })

df_artigo = pd.DataFrame(resultados_artigo)
print(df_artigo.to_markdown(index=False))

# Substituir o modelo antigo pelo novo
joblib.dump(pipeline, 'modelo_ciur_xgboost_final.pkl')
print("\nArquivo 'modelo_ciur_xgboost_final.pkl' foi atualizado com sucesso!")
```

```python
import matplotlib.pyplot as plt
from sklearn.metrics import roc_curve, auc
import seaborn as sns
import pandas as pd

print("\n=== FASE 3: GERANDO GRÁFICOS PARA O ARTIGO ===")

# ---------------------------------------------------------
# 1. PLOTAR A CURVA ROC
# ---------------------------------------------------------
fpr, tpr, thresholds = roc_curve(y_test, probs)
roc_auc = auc(fpr, tpr)

plt.figure(figsize=(8, 6))
plt.plot(fpr, tpr, color='darkorange', lw=2, label=f'XGBoost (AUC = {roc_auc:.4f})')
plt.plot([0, 1], [0, 1], color='navy', lw=2, linestyle='--')
plt.xlim([0.0, 1.0])
plt.ylim([0.0, 1.05])
plt.xlabel('Taxa de Falsos Positivos (1 - Especificidade)', fontsize=12)
plt.ylabel('Taxa de Verdadeiros Positivos (Sensibilidade)', fontsize=12)
plt.title('Curva ROC - Previsão de CIUR', fontsize=14, fontweight='bold')
plt.legend(loc="lower right", fontsize=12)
plt.grid(alpha=0.3)

# Salvar a imagem em alta resolução (300 dpi é o padrão exigido por revistas médicas)
plt.savefig('curva_roc_ciur.png', dpi=300, bbox_inches='tight')
plt.close()
print("-> Gráfico da Curva ROC salvo como 'curva_roc_ciur.png'")

# ---------------------------------------------------------
# 2. PLOTAR A IMPORTÂNCIA DAS VARIÁVEIS (FEATURE IMPORTANCE)
# ---------------------------------------------------------
# Extrair o modelo XGBoost de dentro do pipeline
xgb_model = pipeline.named_steps['xgb']

# Obter a importância das features calculada pela IA
importancias = xgb_model.feature_importances_

# Criar um DataFrame para vincular o nome da coluna com o seu peso
df_importancias = pd.DataFrame({
    'Variável': X.columns,
    'Importância': importancias
})

# Ordenar da mais importante para a menos importante
df_importancias = df_importancias.sort_values(by='Importância', ascending=False)

# Plotar o gráfico de barras horizontais
plt.figure(figsize=(10, 6))
sns.barplot(x='Importância', y='Variável', data=df_importancias, palette='viridis')
plt.title('Importância das Variáveis (Feature Importance) - XGBoost', fontsize=14, fontweight='bold')
plt.xlabel('Impacto Relativo na Decisão do Modelo', fontsize=12)
plt.ylabel('Variáveis Clínicas / Biométricas', fontsize=12)
plt.grid(axis='x', alpha=0.3)

# Salvar a imagem em alta resolução
plt.savefig('importancia_variaveis_ciur.png', dpi=300, bbox_inches='tight')
plt.close()
print("-> Gráfico de Importância das Variáveis salvo como 'importancia_variaveis_ciur.png'\n")

print("Visualizações concluídas com sucesso! Verifique a pasta do projeto.")
```

