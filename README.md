# Pipeline de dados — Olist

## Integrantes

Natan Hideaki Higuti, Breno Kenedy Sato Jacobi, Victor Hugo Allebrandt Freitag,
Renato Augusto Sche e Kaique venandi Moreira.

O roteiro pede grupos de três ou quatro integrantes. O professor já autorizou
a composição com cinco, conforme confirmação do grupo.

## Perguntas de negócio

1. Quais categorias têm o maior valor de produtos vendidos em pedidos entregues, por mês?
2. Como evoluem mensalmente o valor dos produtos vendidos e a quantidade de pedidos entregues?

Valor dos produtos é a soma de `price`, sem frete. Usamos o mês da compra e
pedidos com status `delivered`. Os pedidos são contados por identificador distinto.
Um pedido com várias categorias aparece em mais de uma categoria; não podemos
somar essas contagens para obter o total mensal.

## Fontes

Olist, em três CSVs públicos, sem cadastro:

- https://huggingface.co/datasets/bulutttt/olist-raw-data/resolve/main/olist_orders_dataset.csv
- https://huggingface.co/datasets/bulutttt/olist-raw-data/resolve/main/olist_order_items_dataset.csv
- https://huggingface.co/datasets/bulutttt/olist-raw-data/resolve/main/olist_products_dataset.csv

O segundo candidato foi Chinook. A amostra é baixada deste banco público:
https://raw.githubusercontent.com/lerocha/chinook-database/master/ChinookDatabase/DataSources/Chinook_Sqlite.sqlite

Os tamanhos, nulos e resultados reais ficam nas saídas do notebook. Os downloads
podem mudar ou ficar indisponíveis. A carga principal usa as URLs; os testes de
reconstrução usam cópias locais congeladas na mesma execução.

## Preparação do ambiente

Foi usado Python 3.13.7. Em Windows, na pasta do projeto:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

Abra `pipeline-olist.ipynb` no VS Code com as extensões Python e Jupyter.
Selecione o interpretador `.venv/Scripts/python.exe` como kernel.
O VS Code pode abrir pastas baixadas em modo restrito; revise a pasta e habilite
a confiança manualmente se reconhecer o projeto.

Não é necessário ativar o ambiente no PowerShell nem mudar a política de execução.

## Ordem de execução

Execute as células de cima para baixo, a partir da pasta do projeto.

- Blocos 1 e 2: ambiente, comparação dos candidatos e diagnóstico das fontes.
- Bloco 3: `extract(destino_bronze)` e verificações dos arquivos gravados.
- Bloco 4: `transform(origem_bronze, destino_silver)`.
- Blocos 5 e 6: `load(origem_silver, destino_gold)`, Golds e modelo dimensional.
- Bloco 7: `pipeline()` e testes de dependência, reconstrução e repetição.
- Bloco 8: carga incremental e teste de correção de registro.
- Blocos 9 e 10: gráficos e teste de outro período.
- Bloco 11: conferência da entrega e pendências externas.

Depois de definir as funções, `pipeline()` reconstrói a carga completa em `data`.
A execução não depende de variáveis da exploração dentro das funções principais.
As saídas de teste em `validacao` ficam separadas da carga principal.

## Configuração

`CONFIG`, no início, centraliza caminhos, fontes, colunas, nomes de arquivos,
status, período e fração da carga inicial. O início do período é inclusivo e o fim
é exclusivo. Sem início e fim, usamos todo o período disponível.

O teste do bloco 10 usa janeiro a março de 2018 e restaura a configuração original.
Os parâmetros de cada chamada ficam em `pipeline.log` e `historico_execucoes.csv`.

## Camadas e decisões

### Bronze

`data/bronze`: pedidos, itens e produtos em Parquet, apenas com as colunas
necessárias, mais `_origem` e `_extraido_em` em UTC. Os campos originais são lidos
como texto; campos vazios permanecem vazios. Não há limpeza, filtro ou junção.
A carga completa substitui os mesmos arquivos; não acumula linhas.

### Silver

`data/silver/itens_pedidos.parquet`: um item de um pedido entregue.

- Chave: `order_id + order_item_id`, validada antes e depois da conversão de tipo.
- Cópias idênticas são removidas; conflitos da mesma chave no mesmo lote interrompem a etapa.
- Chaves, status e datas de compra ausentes ou inválidos interrompem o processamento.
- Categoria ausente é preenchida com `sem_categoria`, preservando o valor do item.
- O preço deve ser não negativo e ter até duas casas decimais. Armazenamos também centavos inteiros.
- Junções usam `many_to_one`, exigem correspondência e preservam quantidade e soma dos itens.
- O filtro de status e período vem depois dos joins, com seu efeito contado separadamente.
- Auditoria por regra em `auditoria.csv` e totais em `resumo.json`.

As contagens da Bronze somam fontes com grãos diferentes; não devem ser comparadas
à Silver como se todas as linhas da Bronze fossem itens.

### Gold e modelo dimensional

`data/gold` contém:

- `vendas_categoria_mes.parquet`: categoria e mês, valor e pedidos distintos.
- `vendas_mes.parquet`: mês, valor e pedidos distintos calculados diretamente da Silver.
- `dim_tempo.parquet`: data, ano, mês, trimestre e dia da semana (segunda=0).
- `dim_produto.parquet`: produto e categoria, com chave substituta.
- `fato_vendas.parquet`: item do pedido, chaves das dimensões e valor em centavos.
- `olist.sqlite`: as duas Golds e o modelo dimensional, com PKs e FKs explícitas.
- `graficos/categorias_mes.png` e `graficos/evolucao_mensal.png`.

As Golds usam `if_exists='replace'`. O modelo é reconstruído junto em uma transação:
as chaves podem mudar entre versões, mas fato e dimensões permanecem consistentes.
Categorias corrigidas reclassificam o histórico (tipo 1); não implementamos tipo 2.

A fato contém os identificadores do grão como dimensões degeneradas, sem copiar
textos de categoria ou status. As somas usam inteiros para evitar diferenças de
ponto flutuante. Comparamos SQL e pandas, o JOIN dimensional e a Gold.

O gráfico por categoria mostra as cinco de maior valor no período. O ranking
completo por mês permanece na tabela; o gráfico é um recorte para leitura.

## Reconstrução e carga incremental

O teste de reconstrução renomeia uma pasta de teste e inicia a mesma raiz vazia.
Compara todas as tabelas Parquet e verifica integridade do banco e gráficos.
A repetição desconsidera somente metadados de origem e horário.

O incremental separa pedidos por data em aproximadamente 80% e 20%, mantendo
itens e produtos relacionados. `extract_lote` grava diretórios imutáveis e um
manifesto ordenado. Repetir um identificador exige o mesmo conteúdo. Uma correção
usa outro lote e a versão posterior substitui a anterior por chave.

O Extract completo recusa a Bronze com manifesto para não sobrescrever os lotes.
A Silver e as Golds são recalculadas integralmente; somente a captura ocorre por
lotes. O teste compara o resultado final com a carga completa e testa uma correção
de R$ 1,00 sem criar outro item. Essa correção fica apenas em `validacao`.

Não há tratamento de exclusões por tombstone nesta atividade.

## Logs e histórico

`pipeline.log` usa INFO e ERROR, registra parâmetros e causas de falhas.
`historico_execucoes.csv` guarda etapa, horário UTC, duração, linhas, estado,
mensagem e parâmetros. As linhas escritas no Load contam registros lógicos das
cinco tabelas, sem somar novamente a cópia em SQLite.

Uma etapa com erro registra zero nas contagens por não ter concluído, sem alegar
que isso representa a leitura parcial. Exceções interrompem as etapas seguintes.
Arquivos antigos podem permanecer em disco; consulte o estado da última execução.
A publicação de todos os arquivos de todas as camadas não é uma transação única.

## Entrega e pendências

Entregar notebook executado, README, requirements e `data` com as três camadas.
Log e histórico acompanham as evidências. `validacao` é material de teste e não
substitui `data`. A pasta `.venv` não precisa ser entregue.

A revisão por outro grupo ainda não ocorreu. As respostas correspondentes estão
marcadas como pendentes. A composição com cinco integrantes já foi autorizada pelo professor.
