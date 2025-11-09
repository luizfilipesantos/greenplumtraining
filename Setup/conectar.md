## Conectar no Servidor

**No seu PowerShell (Windows):**
```powershell
ssh -L 5432:localhost:5432 gpadmin@179.185.2.116
```
*Verifique a senha


## Conectar no Banco de Dados

```bash
# Exemplo
psql -h localhost -p 5432 -U <usuario> -d db_<usuario>
```
*Quando pedir senha, digite a sua senha do banco.
