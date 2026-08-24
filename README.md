# Data Preparation Pipeline for Set-Piece Analysis

## Descrição

Este projeto contém um pipeline em Python para preparação e transformação de dados de eventos de futebol.

O principal objetivo do código é converter dados event-level em tabelas estruturadas e adequadas para análise em Power BI, com especial atenção a:

- remates;
- fases de jogo;
- bolas paradas;
- sequências associadas a bolas paradas;
- localização e direção das ações;
- produção ofensiva através de remates e xG.

Os ficheiros CSV gerados pelo pipeline foram posteriormente utilizados para construir as visualizações e análises em Power BI.

---

## Estrutura do pipeline

O pipeline executa, de forma geral, os seguintes passos:

1. leitura dos ficheiros `.pkl`;
2. combinação dos vários jogos numa única tabela de eventos;
3. limpeza e preparação dos principais campos;
4. conversão das coordenadas para um campo de 105 × 68 metros;
5. criação de dimensões de jogos e jogadores;
6. identificação e classificação das bolas paradas;
7. agregação das bolas paradas ao nível de `setPieceId`;
8. criação de uma tabela de remates;
9. criação de uma tabela event-level para análise detalhada das sequências de bola parada;
10. exportação das tabelas finais em formato CSV.

---

# Outputs

O pipeline gera cinco ficheiros principais:

```text
dim_match.csv
dim_player.csv
fact_shots.csv
fact_set_pieces.csv
fact_set_piece_events.csv
```

Estes ficheiros foram utilizados como modelo de dados para a construção das visualizações em Power BI.

---

## `dim_match`

Tabela com uma linha por jogo.

Inclui informação de contexto como:

- `matchId`;
- adversário;
- equipa da casa;
- equipa visitante;
- local do jogo;
- data;
- ordem cronológica.

Esta dimensão permite utilizar filtros e comparar métricas entre diferentes jogos.

---

## `dim_player`

Tabela com uma linha por jogador.

O principal identificador utilizado é o `playerId`.

Inclui informação como:

- nome;
- equipa;
- posição;
- lado da posição;
- label de apresentação.

O ID do jogador é utilizado para evitar ambiguidades entre jogadores com nomes iguais.

---

## `fact_shots`

Tabela com uma linha por remate.

Os penáltis são excluídos da análise.

A tabela inclui, entre outros:

- jogador;
- equipa;
- adversário;
- localização do remate;
- xG;
- post-shot xG;
- resultado;
- fase de jogo;
- classificação do contexto ofensivo.

Os remates são classificados em diferentes padrões de jogo, incluindo:

- `In possession`;
- `Attacking transition`;
- `Second ball`;
- `Corner`;
- `Attacking free kick`;
- `Attacking throw-in`;
- `Other set piece`.

A classificação nativa `SECOND_BALL` é preservada e não é automaticamente incorporada na bola parada anterior.

---

# Bolas paradas

A principal unidade de análise das bolas paradas é o `setPieceId`.

Todos os eventos associados ao mesmo `setPieceId` são tratados como pertencentes à mesma sequência.

A ordenação cronológica dentro da sequência é feita através de `eventNumber`.

---

## Classificação das bolas paradas ofensivas

O pipeline identifica diferentes tipos de reinício e cria uma classificação específica para análise.

### Cantos

Todos os cantos são mantidos.

### Livres ofensivos

São considerados ofensivos os livres iniciados nos últimos 40% do campo:

```text
start_x >= 63
```

### Lançamentos ofensivos

São considerados os lançamentos laterais iniciados no último terço.

É utilizada prioritariamente a classificação nativa da posição no campo:

```text
FINAL_THIRD
OPPONENT_BOX
```

com fallback através das coordenadas:

```text
start_x >= 70
```

As situações são classificadas pela localização do reinício e não pelo resultado posterior da ação.

---

# `fact_set_pieces`

Esta tabela contém uma linha por `setPieceId`.

Cada linha resume uma sequência de bola parada.

Inclui informação sobre:

- jogo;
- equipa atacante;
- tipo de bola parada;
- jogador responsável pelo reinício;
- posição inicial;
- destino da primeira ação;
- características da execução;
- contactos;
- produção ofensiva.

---

## Localização inicial

A posição inicial é retirada do evento de reinício.

Principais campos:

```text
start_x
start_y
start_pitch_position
start_lane
```

---

## Destino da primeira ação

O pipeline utiliza também informação sobre o destino do primeiro evento da sequência:

```text
end_x
end_y
end_pitch_position
end_lane
end_zone
```

Estes campos permitem utilizar a mesma estrutura para estudar diferentes tipos de bola parada, como cantos, livres e lançamentos laterais.

---

## Eventos ofensivos na sequência

O campo:

```text
n_attacking_events
```

representa o número de eventos da sequência realizados pela equipa que executa a bola parada.

Este campo permite distinguir, por exemplo, sequências mais diretas de sequências que envolvem várias ações ofensivas após o reinício.

---

## Produção da bola parada

A tabela inclui métricas como:

```text
shots
xg
goals
shot_created
pxt_attack
pxt_setpiece
```

Para estas métricas é preservada a fase nativa `SET_PIECE`.

Isto significa que uma ação posteriormente classificada como `SECOND_BALL` não é automaticamente adicionada à produção imediata da bola parada.

---

# `fact_set_piece_events`

Esta tabela mantém os eventos individuais associados a cada `setPieceId`.

Enquanto `fact_set_pieces` apresenta uma linha agregada por bola parada, esta tabela permite reconstruir as sequências evento a evento.

Inclui informação como:

- `setPieceId`;
- `eventNumber`;
- jogador;
- tipo de ação;
- ação;
- resultado;
- fase;
- posição inicial;
- posição final;
- lane inicial e final;
- packing zone inicial e final;
- recetor;
- xG;
- pXT;
- informação relativa aos contactos.

Esta tabela permite analisar sequências como:

```text
Reinício
→ Receção
→ Passe
→ Duelo
→ Segunda ação
→ Remate
```

---

## Ordem dos eventos

Os eventos são ordenados através de `eventNumber`.

O pipeline inclui informação que permite identificar a posição de cada evento dentro da sequência.

Desta forma, é possível estudar não apenas o resultado final da bola parada, mas também o mecanismo utilizado até chegar ao remate.

---

# Diferença entre as duas tabelas de bolas paradas

## `fact_set_pieces`

Utilizada para análise agregada:

- número de bolas paradas;
- tipo;
- batedor;
- posição inicial;
- destino;
- remates;
- xG;
- recorrência.

## `fact_set_piece_events`

Utilizada para análise detalhada das sequências:

- ordem das ações;
- jogadores envolvidos;
- movimentos da bola;
- zonas;
- ações intermédias;
- remates e xG gerados durante a sequência.

As duas tabelas foram utilizadas de forma complementar no Power BI.

---

# Coordenadas

As coordenadas ajustadas são convertidas para um campo de:

```text
105 × 68 metros
```

através de:

```python
pitch_x = adjusted_x + 52.5
pitch_y = adjusted_y + 34
```

Não é aplicada normalização min-max.

---

# Utilização em Power BI

Os CSV gerados pelo pipeline foram carregados no Power BI e utilizados para construir o modelo de análise.

A separação entre dimensões e fact tables permite:

- filtrar por jogo;
- filtrar por jogador;
- comparar contextos ofensivos;
- analisar remates e xG;
- estudar bolas paradas;
- reconstruir sequências específicas;
- analisar padrões recorrentes.

A tabela agregada `fact_set_pieces` foi utilizada principalmente para análise de frequência, localização e output das bolas paradas.

A tabela `fact_set_piece_events` foi utilizada quando foi necessário aprofundar as sequências e perceber como determinadas situações evoluíram até ao remate ou à criação de perigo.

---

# Execução

Os ficheiros `.pkl` devem ser colocados numa pasta local `data/`.

Exemplo de estrutura:

```text
project/
│
├── codigo.py
│
├── README.md
│
├── data/
│   ├── match_1.pkl
│   ├── match_2.pkl
│   └── ...
│
└── powerbi_data/
```

No código podem ser utilizados caminhos relativos:

```python
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent

DATA_FOLDER = BASE_DIR / "data"
OUTPUT_FOLDER = BASE_DIR / "powerbi_data"

OUTPUT_FOLDER.mkdir(
    parents=True,
    exist_ok=True
)
```

Para executar:

```bash
codigo.py
```

Após a execução, os ficheiros CSV são gerados na pasta:

```text
powerbi_data/
```

e podem ser importados diretamente para Power BI.
