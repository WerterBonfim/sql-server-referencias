

```sql
select
    total_physical_memory_kb / 1024.0 as MemoriaTotal,
    available_physical_memory_kb / 1024 as MemoriaDisponivel
from sys.dm_os_sys_memory

```
