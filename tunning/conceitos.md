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


