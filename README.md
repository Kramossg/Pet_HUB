# PetHub API

Back end em **Node.js + Express + MySQL** para o banco `pethub_db`.

## Como rodar

```bash
npm install
cp .env.example .env        # ajuste usuário/senha do MySQL e o JWT_SECRET
mysql -u root -p < schema.sql      # cria o banco e as tabelas originais
mysql -u root -p < migration.sql   # adiciona `papel` (Tutor) e `id_responsavel` (Prestador)
npm run dev
```

Para criar um **admin** (homologa prestadores e julga contestações), cadastre-se normalmente e rode:

```sql
UPDATE Tutor SET papel = 'admin' WHERE email = 'seu@email.com';
```
Depois faça login de novo para receber um token com o papel atualizado.

## Autenticação
`Authorization: Bearer <token>` (token vem de `/auth/registrar` ou `/auth/login`).

## Endpoints

| Método | Rota | Quem | O que faz |
|---|---|---|---|
| POST | `/auth/registrar` | público | Cadastra tutor (valida CPF) |
| POST | `/auth/login` | público | Login, retorna JWT |
| GET/PUT | `/auth/me` | logado | Ver / editar meu perfil |
| POST | `/pets` | tutor | Cria pet (gera `codigo_compartilhamento`) |
| GET | `/pets` | tutor | Lista meus pets |
| POST | `/pets/vincular` | tutor | Vincula pet existente pelo código |
| GET/PUT/DELETE | `/pets/:id` | tutor do pet | Detalhe / edita / desvincula |
| GET | `/prestadores?q=&servico=&pagina=&limite=` | público | Busca (só aprovados; impulsionados primeiro, depois relevância) |
| POST | `/prestadores` | logado | Cadastra prestador (fica `Pendente`) com matriz e contatos |
| GET | `/prestadores/meus/lista` | logado | Prestadores sob minha responsabilidade |
| GET/PUT | `/prestadores/:id` | público / dono | Detalhe / edita |
| PATCH | `/prestadores/:id/homologacao` | admin | Aprova/reprova e define taxa |
| POST | `/prestadores/:id/servicos` · `PUT/DELETE /servicos/:id` | dono | Catálogo de serviços |
| POST | `/prestadores/:id/enderecos` · `DELETE /enderecos/:id` | dono | Matriz/filiais |
| POST | `/prestadores/:id/contatos` · `DELETE /contatos/:id` | dono | Canais de contato |
| POST/GET | `/prestadores/:id/impulsionamentos` | dono | Impulsiona (Diario=1d, Semanal=7d, Mensal=30d) |
| PATCH | `/impulsionamentos/:id/cancelar` | dono | Cancela impulsionamento |
| POST | `/atendimentos` | tutor | Solicita atendimento `{id_prestador, id_pet}` |
| GET | `/atendimentos` | tutor | Meus atendimentos |
| PATCH | `/atendimentos/:id` | dono do prestador | Atualiza `status_agenda` / `status_compatibilidade` |
| GET | `/prestadores/:id/atendimentos` | dono | Atendimentos recebidos |
| POST | `/atendimentos/:id/avaliacao` | tutor | Avalia (1–5), só se `Confirmado` |
| GET | `/prestadores/:id/avaliacoes` | público | Avaliações do prestador |
| POST | `/avaliacoes/:id/contestar` | dono do prestador | Contesta → `Em Analise` |
| PATCH | `/avaliacoes/:id/contestacao` | admin | `{decisao: "Deferida"\|"Indeferida"}` |
| GET | `/avaliacoes/contestacoes/pendentes` | admin | Fila de contestações |

## Regras de negócio implementadas
- `indice_relevancia` = média das notas, **ignorando avaliações com contestação deferida**; recalculado a cada avaliação/julgamento.
- Só prestador **Aprovado** aparece na busca, recebe atendimentos e pode impulsionar.
- Um atendimento só pode ser avaliado uma vez (UNIQUE no banco) e só se estiver `Confirmado`.
- Vários tutores podem compartilhar o mesmo pet; ao remover o último tutor, o pet é apagado (a não ser que já tenha histórico de atendimentos).

## Exemplo rápido
```bash
curl -X POST localhost:3000/auth/registrar -H 'Content-Type: application/json' -d '{
  "nome_completo":"Ana Souza","data_nascimento":"1990-05-10","cpf":"529.982.247-25",
  "endereco":"Rua das Flores, 100, Joinville","email":"ana@email.com","senha":"senha1234"}'

curl -X POST localhost:3000/pets -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"nome":"Thor","sexo":"Macho","especie":"Cachorro","raca":"Labrador","idade":3}'
```

## Pendências (não estavam no schema)
- Pagamento do impulsionamento e da taxa de permanência.
- Lógica real de `status_compatibilidade` (hoje é definida manualmente pelo prestador/admin).
- Upload de foto/documentos (hoje só guarda a URL).
