## O que é pagina de dados

- Uma página de dados é a menor alocação de dados utilizado pelo SQL Server.
Ela é a uniade fundamental de armazenamento de dados.

* Definido de 8Kbytes ou 8192 bytes que são divididos em cabeçalho, área de dados e slot de controle.

* Uma página de dados é exclusiva para um objeto de alocação e um objeto de alocação pode ter diversas páginas de dados.

* Em uma página de dados somente serão armazenados 8060 bytes de dados.


```sql 
-- Exibe o número de linhas, o espaço em disco reservado e o espaço em 
-- disco usado por uma tabela, exibição indexada ou fila Agente de Serviço 
-- no banco de dados atual ou exibe o espaço em disco reservado e usado 
-- pelo banco de dados inteiro.
execute sp_spaceused 'Teste01'
```

Referencias: 

SQL Server - Uma visão da Storage Engine.
https://www.youtube.com/watch?v=1-qdF-txeAQ



### Extent ou Extensão.

* São agrupamentos lógicos de página de dados.
* Se objetivo é gerenciar melhor espaço alocado dos dados.
* Um Exetent tem exatamente 8 páginas de dados e um tamanho de 64 kbytes.

### Extents podem ser
* Misto (Mixed Extent), quando as páginas de dados são objetos de alocações.
* Uniforme (Uniform Extent), quando as páginas de dados são exclusiva de um único objeto de alocação.



```sql
select
    total_physical_memory_kb / 1024.0 as MemoriaTotal,
    available_physical_memory_kb / 1024 as MemoriaDisponivel
from sys.dm_os_sys_memory

-- Analisa a quantidade de memoria
select db_name(database_id)            as BancoDeDados,
       (count(1) * 8192) / 1024 / 1024 as nQtdPaginas
from sys.dm_os_buffer_descriptors
group by db_name(database_id)
order by nQtdPaginas desc

```

## Design de banco de dados - Criando um banco com varios arquivos



```sql

select * from sys.database_files

```

* Cada arqauivo tem um FILE ID que é o número de indentificação do arquivo.
* A coluna DATA_SPACE_ID é a identificação desse arquivo dentro de um grupo
de arquivo.
* A coluna NAME é o nome lógico do arquivo.
A coluna SIZE é o tamanho alocado do arquivo em páginas de dados.
* A oluna GROWTH é a taxa de crescimento do arqivo em página de dados.



```sql

select ([size] * 8192) / 1024 as nTamanhoKB,
       (growth * 8192) / 1024 as nCrescimentoKB
  from sys.database_files

```




```sql

drop database if exists DBDemoA
go

create database DBDemoA
on primary                                                  -- FG Primary.
(
    name = 'Primario',                                      -- Nome lógico do arquivo
    filename = '/var/opt/mssql/data/DBDemoA_Primario.mdf',  -- Nome físico do arquivo
    size = 256MB,                                           -- Tamanho inicial do arquivo.
    filegrowth = 64mb                                       -- Valor de crescimento
    )
log on (
    name = 'Log',
    filename = '/var/opt/mssql/data/DBDemoA_Log.ldf',
    size = 12mb,
    filegrowth = 8mb
    )

go


```

FILEGROUP
---------

- FILEGROUP é um agrupamento lógico de arquivos de dados para distribuir melhor a 
alocação de dados entre os discos, agrupar dados de acordo com contextos ou 
arquivamentos como também permitir ao DBA uma melhor forma de administração.

No nosso caso, vamos focar em melhorar o desempenho das consultas.




A query abaixo cria um banco primario onde é armazenado os metadatas,
dois bancos (FG dados) onde são armazenado os dados e um banco de log.

É aconcelhavel quebrar um Filegroup dois ou mais arquivos, dependendo da demanda de acesso ao banco de dados


```sql
drop database if exists DBDemoA
go



create database DBDemoA
    on primary  -- Metadados
    (
        name = 'Primario',
        filename = '/var/opt/mssql/data/DBDemoA_Primario.mdf',
        size = 64mb,
        filegrowth = 8mb
    ),
    filegroup DADOS
    (
        name = 'DadosTransacional1',
        filename = '/var/opt/mssql/data/DBDemoA_SecundarioT1.ndf',
        size = 1024mb,
        filegrowth = 1024mb
    ),
    (
        name = 'DadosTransacional2',
        filename = '/var/opt/mssql/data/DBDemoA_SecundarioT2.ndf',
        size = 1024mb,
        filegrowth = 1024mb
    ),
    filegroup INDICES
    (
        name = 'IndicesTransacionais1',
        filename = '/var/opt/mssql/data/DBDemoA_I1.ndf',
        size = 1024mb,
        filegrowth = 1024mb,
        maxsize = 10gb
        ),
    (
        name = 'IndicesTransacionais2',
        filename = '/var/opt/mssql/data/DBDemoA_I2.ndf',
        size = 1024mb,
        filegrowth = 1024mb,
        maxsize = 10gb
        ),
    filegroup DADOSHISTORICO
    (
        name = 'DadosHistorico1',
        filename = '/var/opt/mssql/data/DBDemoA_SecundarioH1.ndf',
        size = 1024mb,
        maxsize = 20gb
        ),
      (
        name = 'DadosHistorico2',
        filename = '/var/opt/mssql/data/DBDemoA_SecundarioH2.ndf',
        size = 1024mb,
        maxsize = 20gb
        )
log on
    (
        name = 'Log',
        filename  = '/var/opt/mssql/data/DBDemoA_Log.ldf',
        size  = 512mb,
        filegrowth = 64mb
    )

go


-- Estamos dizendo para o SQL SERVER em qual Filegroup padrão os
-- objetos de alocação de dados (tabelas, por exemplo) serão criados.
alter database [DBDemoA] modify filegroup [DADOS] default
go


-- Query mais completa sobre o ambiente

select (df.size * 8192) / 1024 as nTamanhoKB,
       (df.growth * 8192) / 1024 as nCrecimentoKB,
       df.name as cNomeLogico,
       df.physical_name as cNomeFisico,
       ds.name as cFileGroup,
       ds.type as cTipoFileGroup,
       ds.type_desc,
       ds.is_default
from sys.database_files as df
left join sys.data_spaces as ds
    on df.data_space_id = ds.data_space_id


```

## Tipos de dados e armazenamento


CHAR(n)     - Tipo de dado caracter que aceita 'n' bytes. O total de bytes
              declarado no tipo do dados será o mesmo para o armazenamento, independente
              da quantidade de caracteres associado. 
              Em um CHAR(10), por exemplo, mesmo que você inclua a palavra 
              'JOSE' (4 bytes) o SQL SERVER grava 10 bytes.

              
NCHAR(n)    - Tipo de dado UNICODE que aceita 'n' bytes, mas armazena 2*n bytes.
              ele utiliza 2 bytes para representar um caracter.
              A palavra 'JOSE' em um tipo NCHAR(10) será gravado com 20 bytes de armazenamento.
              O dados deve ser representado com o N maiúsculo na frente do litera.


```sql
use DBDemo
go

go
drop table if exists Teste
go

Create Table Teste 
(
   Nome char(20),
   NomeInt nchar(20) 

)

insert into Teste (Nome, NomeInt) values ('JOSE DA SILVA',  'ホセ ダ シルヴァ')
insert into Teste (Nome, NomeInt) values ('JOSE DA SILVA', N'ホセ ダ シルヴァ')

select * from teste




declare @Nome1 varchar(10) = 'Jose'
select len(@Nome1) , datalength(@Nome1)
 
declare @Nome2 nvarchar(10) = N'Jose'
select len(@Nome2) , datalength(@Nome2)


/*
Algumas boas práticas 

1. Não use NCHAR ou NVARCHAR.

2. Utiliza INT para CHAVE PRIMARIA das tabelas, armazenar valores inteiros acima de 32.767.
   Lembre-se que mesmo gravando valores pequenos como 10, 50 ou 100, o armazenamento será de 
   4 bytes. 

3. Se a tabela armazenará no máximo 30.000 linhas e se voce tem controle dessa quantidade
utilize SMALLINT como CHAVE PRIMARIA.

4. Pequenas tabelas para armazenar categorias, grupos ou tipificação, veja a possibilidade 
   de usar TINYINT para identificacação das linhas.

4. Utilize BIGINT somente em caso de real necessidade. 

5. Utilizar VARCHAR somente para colunas com variações grandes de dados e com tamanhos grandes.

   VARCHAR(10) será armazenado 12 bytes, 20% do espaço é de controle.
   Se voce utilizar esse tipo de dados para cadastrar o telefone fixo sem DDD, por exemplo:
      
   27999999 --> São 8 bytes

   Tipo           Armazenamento 
   -------------- -------------
   VARCHAR(10)               10 
   CHAR(10)                  10   
   NCHAR(10)                 16
   NVARCHAR(10)              18

6. Analise o uso de CHAR ou INT para representar números. 

7. Armazenar valores que são resultado de cálculos de outras colunas, somente se voce utiliza
   essa coluna como pesquisa.

   Create Table ItemPedido
   (
      IdItem int,
      idProduto int ,
      nQuantidade smallint ,
      nPrecoUnitario smallmoney,
      nPrecoTotal smallmoney
   )

   A coluna nPrecoTotal será obtida com a operação nQuantidade * nPrecoUnitario. Então não
   precisa armazenar esse dados na coluna nPrecoTotal.

   Create Table ItemPedido
   (
      IdItem int,
      idProduto int ,
      nQuantidade smallint ,
      nPrecoUnitario smallmoney,
      nPrecoTotal as nQuantidade * nPrecoUnitario
   )

8. Utilize DATE quando deseja registrar um evento com dia mes e ano. 
   Exemplo: Data de construção, Data de nascimento

*/
```


```sql

-- Cria uma tabela e define seu FileGroup

create table exemplo02
(
    id int,
    nome char(20)
) on Dados2 -- Definição do FG


insert into tabela () values ()
go 15000 -- executa 15 mil vezes o insert


```

## Melhores praticas na hora de criar um tabela

* Usar varchar ao inves de nvarchar
* Quando possivel, usar smallint
* Quando possivel, usar smallmoney,
* Quando possivel, usar coluna computada: ValorComissao as (Preco * Comissao / 100)
* Particionar as tabelas em Filegroups que façam sentido
* Quando criar uma tabela, crie com a menor quantidade possivel de paginas

## Colunas calculadas

Uma coluna calculada é utilizada quando realizamos um cálculo ou montamos uma 
expressão e associamos a uma coluna.

```sql

create table examplo(
    preco   smallmoney,
    comissao    numeric(5,2),
    valorComissao (preco * comissao / 100), -- ocupara 13 bytes
    valorComissao2 cast((preco * comissao / 100) as smallmoney), -- opara 4 bytes
    valorComissao3 cast((preco * comissao / 100) as smallmoney), persisted -- persistirar no disco 
)

```

Em alguns momentos não é viavel ter uma coluna calculada, deve ser fazer uma analise 


## Compactação de dados

* reduzir o espaço alocado e aumentar a desempenho de acesso aos dados
* Não é toda tabela qu epode ser compactada ou que realmente teremos
algum ganho de armazenamento ou performance se compactamos.

Onde não a ganho:

* Tabel acuja a soma dos bytes armazenados for próximo a 8060 caracteres.
* Tabela que contém muitos dados exclusivos ou únicos.

```sql
-- Gera uma estivativa de compreesão, mais não realiza a mesma.
exec sp_estimate_data_compression_savings 'dbo', '[nome da tabela]', null, null, 'PAGE'
```

Uma taxa acima de 25% é um bom candidato a ter seus dados compactados.

```sql 

-- Mostra as paginas que estão alocadas na tabela
declare @nomeDaTabela as varchar(50) = 'Clientes'
select total_pages, used_pages, data_pages, p.data_compression_desc
from sys.allocation_units au
         join sys.partitions p
              on au.container_id = p.partition_id
where p.object_id = object_id( @nomeDaTabela )
  and au.type = 1

```



## Conhecendo visões de gerenciamento dinâmico - DMV

* Ajudam a indentificar as querys mais lentas.
* Podem ser views ou functions.
* São sempre acessadas através do schema SYS e na grande maioria dos casos,
começam com o prefixo DM


```sql

-- mostra todas as views e functions do tipo DMV
select name, type, type_desc
from sys.system_objects
where name like 'DM[_]%'
order by name

```



## Conceito de BTree.

https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html

* Uma das técnicas mais eficientes para organizar dados para uma pesquisa rápida 
  é a utilização de ordenação utilizando uma estrutura de dados conhecida como árvore binária.

  