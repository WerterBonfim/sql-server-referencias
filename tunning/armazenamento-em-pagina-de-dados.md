## Estrutura de página de dados

* Uma página de dados é a manor alocação de dados utlizada pelo Sql Server.
Ela é a unidade fundamental de armazenamento


```
PÁGINA DE DADOS - 8Kb ou 8192 bytes 

+-----------------------------------------------------------+
|                                                           |
|          CABEÇALHO (HEADER) DA PÁGINA - 96 BYTES          |
|                                                           |
+-----------------------------------------------------------+
|                                                           |
|                       ÁREA DE DADOS                       | 
|                                                           |
|         TAMANHO MÁXIMO DE UMA LINHA : 8060 BYTES          |
|                                                           |
|                                                           |
|                                                           |
+-----------------------------------------------------------+
| MATRIZ DOS SLOTS - 2 BYTES POR LINHA             |  |  |  |
+-----------------------------------------------------------+


Uma página de dados é exclusiva de um objeto de alocação de dados (Tabela ou Índice). 

Cabeçalho      : ID da Página, ID do Objeto, Tipo da Página, espaço live, etc....
Área de Dados  : Onde as linhas serão armazenadas. Alocadas em série, a partir do final do 
                 cabeçalho. Cada linha tem o limite de 8060 bytes. 
Matriz de Slot : Uma tabela que contém para cada linha, a posição que ele se inicia dentro da 
                 página. Também conhecida como tabela de deslocamento de linha ou offset row. 

Considerando a área de dados e matriz de slots, temos 8.096 bytes para armazenamento.

Ref.: 
https://docs.microsoft.com/pt-br/sql/relational-databases/pages-and-extents-architecture-guide

```


### Set Statistics io:

* Quando ligado, uma instrução é executada, o Sql Server apresenta as estatísticas de acesso ao cache ou buffer ou a área de disco.

```sql
set statistics io on -- liga a apresentação
set statistics io off -- deliga a apresentação
```


Quais dados são apresentados: 

Para cada tabela envolvida na instrução, é apresenta uma linha 
com as informações de estatísticas. São elas:

| Propriedade      | Descrição                                                                  |
|------------------|----------------------------------------------------------------------------|
| table           | Nome da tabela                                                             |
| scan count       | Contagem de buscas para recuperar os dados.                                |
| logical reads    | Quantidade de páginas acessadas no Buffer Pool (cache de dados)            |
| physical reads   | Quantidade de páginas acessadas no disco.                                  |
| read-ahead reads | Quantidade de páginas incluídas no Buffer Pool. Chamada leitura antecipada |



Outras informações contidas no resultado são referentes a dados LOB (Large Object ou 
tipo de dados para grandes objetos) como varchar(max) ou varbinary(max). 
São eles lob logical reads, lob physical reads e lob read-ahead reads. 
LOB serão tratados em uma seção específica.


## O que é uma Heap Table

* Uma tabela que não tem índices clusterizado.

Como ela não tem índices para a coluna, o SQL Server
precisar ler toda as páginas da tabela para encontrar a linha que
satisfação o predicado.

Mesmo que voce inclua os dados em uma ordem que deseja que eles fiquem, 
uma Heap Table não tem em sua estrutura os dados algo que indique que esses dados
estão ordenados.

Quando utilizar uma Heap Table:
* Tabelas pequenas com poucas linhas e colunas cuja a soma total de bytes que serão
armazenados for menor que 8060 bytes.

Ref.: https://docs.microsoft.com/pt-br/sql/relational-databases/indexes/heaps-tables-without-clustered-indexes



## Localizando a página de dados de uma linha da tabela.

```sql


select sys.fn_PhysLocFormatter(%%PHYSLOC%%) AS LocalFisico, *
from tmovimento;


```
A coluna LocalFisico é dividia em três partes:
1       - ID do arquvo de dados.
5618    - Número da página de dados. 
49      - ID do Slot (Posição da linha dentro da página)



## Estrutura de um Extent

* Extent ou extenção sáo agrupamentos de 8 páginas de dados
fisicamente contíguas.

* Tem o tamanho de 64k

* O objetivo é gerencias melhor o armazenamento físico dos dados.





```sql 

-- a pouca documentação sobre a function:
-- dm_db_database_page_allocations

select extent_page_id as extent,
       allocated_page_page_id as page,
       is_mixed_page_allocation
from sys.dm_db_database_page_allocations(
    db_id(),
    object_id('TesteExtendA'),
    null,
    null,
    'DETAILED')
order by page


select object_name(object_id) as tabela,
       allocated_page_page_id as pageId
from sys.dm_db_database_page_allocations(
    db_id(),
    null,
    null,
    null,
    'DETAILED')
where extent_page_id = 344
order by pageId
```
