```sql

sp_help '[nome da tabela]'

-- Mostra quanto de armazenamento esta em uso.
exec sp_spaceused '[nome da tabela]'

-- Gera uma estivativa de compreesão, mais não realiza a mesma.
exec sp_estimate_data_compression_savings 'dbo', '[nome da tabela]', null, null, 'PAGE'



-- Mostra as paginas que estão alocadas na tabela
declare @nomeDaTabela as varchar(50) = 'Clientes'
select total_pages, used_pages, data_pages, p.data_compression_desc
from sys.allocation_units au
         join sys.partitions p
              on au.container_id = p.partition_id
where p.object_id = object_id( @nomeDaTabela )
  and au.type = 1




-- Estimativa de compactação

declare @nomeDaTabela as varchar(50) = '[nome da tabela]'
declare @tabela table (
    a varchar(128),
    b varchar(128),
    c int,
    d int,
    [size_with_current_compression_setting(KB)] bigint,
    [size_with_requested_compression_setting(KB)] bigint,
    e bigint,
    f bigint
)
declare @tamanhoAtual as bigint
declare @tamanhoEstimado as bigint
declare @porcentagemDeMelhoria as bigint

insert @tabela (a, b, c, d, [size_with_current_compression_setting(KB)], [size_with_requested_compression_setting(KB)], e, f)
exec sp_estimate_data_compression_savings 'dbo', @nomeDaTabela, null, null, 'PAGE'
select @tamanhoAtual = [size_with_current_compression_setting(KB)],
       @tamanhoEstimado = [size_with_requested_compression_setting(KB)]
from @tabela

set @porcentagemDeMelhoria = 100- ((@tamanhoEstimado * 100) / @tamanhoAtual)
select @tamanhoAtual as tamanhoAtual,
       @tamanhoEstimado as tamanhoEstimado,
       @porcentagemDeMelhoria as porcentagemDeMelhoria



-- mostra todas as views e functions do tipo DMV
select name, type, type_desc
from sys.system_objects
where name like 'DM[_]%'
order by name



-- Cria um view que facilita a busca das DMVs do sql

create or alter view vDMVs
as
select substring(substring(name, 4, 100), 1, charindex('_', substring(name, 4, 100)) -1) as tipo,
       name,
       type,
       type_desc
from sys.system_objects
where name like 'DM[_]%'
go


-- lista as os programas que abriram sessão com sql server
-- as linhas antes de 51 são do proprio sql server
select * from sys.dm_exec_sessions where session_id >= 51


```
