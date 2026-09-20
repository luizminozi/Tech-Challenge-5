# Datathon Passos Mágicos — Fase 5 (POSTECH)

Solução completa para o desafio: limpeza e análise de dados, storytelling
respondendo às 11 perguntas de negócio, modelo preditivo de **risco de
defasagem** e aplicação Streamlit para uso da equipe da Passos Mágicos.

## Estrutura da pasta

```
tech_challenge_5_datathon/
├── README.md                                    <- este arquivo
├── requirements.txt                              <- dependências do app Streamlit
├── 1_analise_exploratoria_e_limpeza.ipynb        <- limpeza + EDA + 11 perguntas (Colab)
├── 2_modelo_preditivo_risco_defasagem.ipynb      <- feature eng. + treino + avaliação (Colab)
├── 3_app_streamlit.py                            <- app Streamlit (usa o modelo treinado)
├── data/
│   ├── BASE_DE_DADOS_PEDE_2024_-_DATATHON.xlsx   <- base bruta (você precisa colocar aqui)
│   └── painel_tratado.csv                        <- gerado pelo notebook 1 (já incluso)
└── modelo/
    └── pipeline_risco_defasagem.joblib           <- modelo treinado (já incluso, pronto pra usar)
```

> Os notebooks 1 e 2 **já foram executados uma vez do início ao fim** contra
> a base real para gerar o `painel_tratado.csv` e o `pipeline_risco_defasagem.joblib`
> que estão nesta pasta — ou seja, você já pode rodar o app Streamlit direto,
> sem precisar rodar os notebooks primeiro. Mas se quiser reproduzir/auditar
> tudo do zero (ou usar dados atualizados no futuro), os notebooks estão
> prontos para isso.

---

## 1. Rodando os notebooks no Google Colab

1. Suba esta pasta inteira (`tech_challenge_5_datathon/`) para o seu Google Drive.
2. Abra `1_analise_exploratoria_e_limpeza.ipynb` no Google Colab
   (clique direito no arquivo no Drive → "Abrir com" → "Google Colaboratory").
3. Rode as células em ordem (`Ambiente de execução` → `Executar tudo`). A
   primeira célula monta o Google Drive — autorize o acesso quando solicitado.
4. Repita para `2_modelo_preditivo_risco_defasagem.ipynb`.

**Ajuste de caminho:** os notebooks assumem que a pasta do projeto está em
`/content/drive/MyDrive/tech_challenge_5_datathon`. Se você organizou de
outro jeito no seu Drive, edite a variável `CAMINHO_PROJETO` na segunda
célula de cada notebook.

**Bibliotecas extras:** o Colab já vem com pandas, numpy, scikit-learn,
matplotlib e seaborn pré-instalados. Os notebooks instalam sozinhos as duas
dependências extras que faltam (`xgboost` e `imbalanced-learn`) via `!pip
install` — não precisa fazer nada manualmente.

**Ordem recomendada:** rode o notebook 1 antes do notebook 2 (ele gera o
`painel_tratado.csv`, o que deixa o notebook 2 mais rápido). Mas não é
obrigatório — o notebook 2 detecta se o CSV não existe e refaz a limpeza
sozinho a partir da planilha bruta.

---

## 2. O que cada notebook faz

### `1_analise_exploratoria_e_limpeza.ipynb`
Limpeza e padronização das 3 abas da planilha (2022/2023/2024), com cada
inconsistência real encontrada explicada passo a passo (nomes de coluna
diferentes, categorias com grafias diferentes, um bug de coluna trocada em
2024, um bug de formatação de data em 2023, placeholders como `"INCLUIR"`).
Depois, responde às 11 perguntas de negócio do enunciado, cada uma com
gráfico + interpretação. Termina exportando `painel_tratado.csv`.

### `2_modelo_preditivo_risco_defasagem.ipynb`
Feature engineering (com justificativa de cada escolha, incluindo por que
`INDE`, `Pedra`, `IAN`, `Fase` e `Defasagem` **não** entram como features —
vazariam o alvo), separação treino/teste por aluno (`GroupShuffleSplit`,
evitando vazamento entre anos do mesmo aluno), comparação de 6 modelos
(Regressão Logística, Árvore de Decisão, Random Forest, Gradient Boosting,
XGBoost e uma Rede Neural/MLP), avaliação do modelo final e exportação do
pipeline treinado para `modelo/pipeline_risco_defasagem.joblib`.

**Resultado:** Gradient Boosting venceu com ROC-AUC ≈ 0,87, superando a rede
neural testada — a análise completa do porquê está no notebook.

---

## 3. Rodando o app Streamlit localmente

```bash
cd tech_challenge_5_datathon
pip install -r requirements.txt
streamlit run 3_app_streamlit.py
```

O app abre em `http://localhost:8501`. Ele tem duas abas: avaliação de um
aluno por vez (formulário) e avaliação em lote (upload de CSV).

---

## 4. Compatibilidade de versões — leia antes de fazer o deploy

Esta seção existe porque testamos e **confirmamos na prática** um jeito real
desse projeto quebrar silenciosamente, então vale a pena entender o porquê
das escolhas abaixo antes de mudar alguma coisa.

### O problema real: o modelo é um objeto `pickle`, não um formato universal

O `3_app_streamlit.py` carrega `modelo/pipeline_risco_defasagem.joblib`, que
é um `Pipeline` inteiro do scikit-learn (imputer + scaler + one-hot encoder +
Gradient Boosting) serializado com `joblib`. Esse formato **não é
compatível entre versões diferentes do scikit-learn**. Testamos isso na
prática: carregar o `.joblib` gerado com scikit-learn **1.8.0** usando
scikit-learn **1.4.2** falha direto com:

```
ModuleNotFoundError: No module named '_loss'
```

Não é um aviso, é uma falha total. Por isso as versões de `scikit-learn`,
`pandas`, `numpy` e `joblib` usadas para **treinar** o modelo (notebook 2)
precisam ser **exatamente as mesmas** usadas para **servir** o modelo
(Streamlit). É esse o motivo de:

- o `requirements.txt` usar `==` (versão travada) em vez de `>=`;
- o notebook 2 ter, na primeira célula de instalação, um `pip install` com
  as mesmas versões exatas do `requirements.txt` — **em vez de confiar no
  que o Colab já vem com por padrão**, que muda com o tempo;
- o notebook 2 imprimir as versões realmente ativas logo depois dos
  imports, para você conferir antes de treinar.

**Se algum dia você atualizar uma versão, atualize as duas pontas juntas**
(a célula de instalação do notebook 2 **e** o `requirements.txt`), retreine
o modelo, e gere um `.joblib` novo.

### Sobre a sua pergunta: Python 3.11 no Streamlit funciona?

Sim — checamos a disponibilidade de pacote (`pip download`, sem precisar
rodar de fato em 3.11) e confirmamos que `numpy==2.4.4`, `pandas==3.0.2` e
`scikit-learn==1.8.0` têm wheel pronta tanto para **Python 3.11** quanto
para **Python 3.12**. Então não há bloqueio técnico em usar 3.11.

Dito isso, o que a gente **testou de ponta a ponta de verdade** (treino no
notebook + carregamento do modelo + previsão) foi em **Python 3.12** — que
também é, hoje, o runtime padrão do próprio Google Colab. Por isso a
recomendação é: **use Python 3.12 no Streamlit também**, para eliminar mais
uma variável (você replica exatamente o ambiente que treinou o modelo). Se
por algum motivo você precisar de 3.11 (ex.: outra dependência do seu
projeto exige), pode usar — só não misture com versões diferentes das
pinadas acima sem retreinar o modelo.

### Como escolher a versão do Python no Streamlit Community Cloud

⚠️ **Não use um arquivo `runtime.txt`** para isso — em vários relatos recentes
(2025-2026) de usuários e no próprio rastreador de issues do Streamlit, o
Community Cloud está **ignorando o `runtime.txt`** e usando a versão mais
recente do Python de qualquer forma, o que quebra dependências como numpy/
pandas/scikit-learn. O jeito que a documentação oficial do Streamlit
recomenda — e que realmente funciona — é escolher a versão **na hora do
deploy**, pela interface:

1. Em [share.streamlit.io](https://share.streamlit.io), clique em **"New app"**.
2. Aponte para o repositório, branch e `3_app_streamlit.py` como de costume.
3. Antes de clicar em "Deploy", abra **"Advanced settings"**.
4. No campo de versão do Python, selecione **3.12** (ou 3.11, se preferir —
   ambos têm wheel disponível para as versões pinadas acima).
5. Só então clique em **"Deploy"**.

Se o app já estiver no ar e você quiser trocar a versão do Python depois,
não dá para editar in-place — é preciso **apagar o app e reimplantar**,
escolhendo a nova versão nas "Advanced settings" novamente (essa é uma
limitação documentada do próprio Streamlit Community Cloud).

---

## 5. Deploy no Streamlit Community Cloud

1. Crie um repositório no GitHub e suba esta pasta inteira, **incluindo**:
   - `3_app_streamlit.py`
   - `requirements.txt`
   - a pasta `modelo/` com o `pipeline_risco_defasagem.joblib`
   (o arquivo de dados brutos/`data/` não precisa ir para o repositório —
   o app só depende do modelo já treinado).
2. Acesse [share.streamlit.io](https://share.streamlit.io) e faça login com
   sua conta GitHub.
3. Clique em **"New app"**, selecione o repositório e a branch.
4. Em **"Main file path"**, aponte para `3_app_streamlit.py`.
5. Abra **"Advanced settings"** e selecione a versão do Python (**3.12**
   recomendado — veja o porquê na seção 4 acima) **antes** de clicar em
   "Deploy".
6. Clique em **"Deploy"**. Em alguns minutos, você terá uma URL pública
   (algo como `https://seu-app.streamlit.app`) para compartilhar com a
   equipe da Passos Mágicos.

Se o modelo for retreinado (novo ciclo de dados), rode o notebook 2
novamente, substitua o arquivo `.joblib` no repositório e **confira se as
versões impressas pelo notebook ainda batem com o `requirements.txt`** (veja
a seção 4) antes de subir a atualização.

---

## 6. Por que Machine Learning (e não Deep Learning)

Testamos empiricamente — não só por teoria — 5 modelos de ML tradicional e 1
rede neural (MLP). Os modelos baseados em árvore (Gradient Boosting e
XGBoost) ficaram no topo (ROC-AUC ≈ 0,87), e a rede neural ficou **atrás**
de todos os modelos de árvore. Isso é esperado para este cenário: poucos
milhares de linhas e ~15 features tabulares é um regime de dados onde
ensembles de árvore tradicionalmente superam redes neurais, que precisam de
volumes bem maiores de dado para mostrar vantagem. Os detalhes completos
(código, métricas, gráficos) estão no notebook 2, seção 5.

---

## 7. Alvo do modelo: por que "risco de defasagem" e não INDE/Pedra

O enunciado pede um modelo que estime a *probabilidade do aluno entrar em
risco de defasagem* (pergunta 9). Durante a exploração dos dados (notebook
1, seção 4), validamos que o **INDE é uma combinação linear exata dos outros
7 indicadores** (fórmula oficial da Passos Mágicos) — ou seja, "prever" o
INDE ou a Pedra usando esses mesmos indicadores não seria aprendizado de
máquina de verdade, seria resolver uma equação (vazamento de dados/data
leakage). Por isso, o modelo prevê diretamente o **risco de defasagem**
(baseado no critério oficial de IAN: `defasagem < 0`), usando como features
apenas os indicadores que **não** compõem essa definição, mais dados de
perfil e histórico do aluno — uma tarefa de previsão genuína.
