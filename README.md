# 📚 Documentação Técnica Completa - TIVIT Academy

## 📑 Sumário Executivo

O **TIVIT Academy** é uma plataforma completa de gestão educacional desenvolvida com tecnologias modernas e escaláveis. O sistema suporta múltiplos perfis de usuário (Aluno, Professor, Administrador) com funcionalidades específicas para cada grupo, incluindo matrícula, gestão de notas, controle de frequência, distribuição de conteúdo e análise de desempenho.

**Stack Técnico:**
- **Backend:** C# .NET 9, ASP.NET Core, Entity Framework Core, SQL Server
- **Frontend:** React 18, TypeScript, Vite, React Router DOM
- **Infraestrutura:** AWS SQS, QuestPDF
- **Validação:** Zod (Schema validation)
- **Autenticação:** JWT (JSON Web Tokens)

---

## 1. Visão Geral da Arquitetura

### Estrutura em Camadas

```
┌─────────────────────────────────────────────────────────┐
│                  TIVIT ACADEMY                           │
├─────────────────────────────────────────────────────────┤
│  FRONTEND (React + TypeScript + Vite)                   │
│  - Pages (Dashboards por perfil)                        │
│  - Components (UI Reutilizável)                         │
│  - Context API (Autenticação Global)                    │
│  - Services (Integração API)                            │
│  - Hooks (Lógica Customizada)                           │
├─────────────────────────────────────────────────────────┤
│  REST API (ASP.NET Core 9)                              │
│  - 13 Controllers REST                                  │
│  - 11+ Services (Business Logic)                        │
│  - 27 DTOs (Data Transfer)                              │
│  - Exception Middleware                                 │
├─────────────────────────────────────────────────────────┤
│  DATA ACCESS (Entity Framework Core)                    │
│  - 15 Models (Entities)                                 │
│  - AppDbContext                                         │
│  - Code First Migrations                                │
│  - SQL Server Database                                  │
├─────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE                                          │
│  - AWS SQS (Message Queue)                              │
│  - QuestPDF (Report Generation)                         │
│  - JWT Token Service                                    │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Backend - ASP.NET Core 9

### 2.1 Estrutura Completa do Backend

**Controllers (13 total):**
- AlunoController
- ChamadaController
- ConteudoController
- CursoController
- EventoController
- LoginController
- MateriaController
- MatriculaController
- NotaController
- NotificacaoController
- ProfessorController
- TurmaController
- UserController

**Services (11+ total):**
- MatriculaService
- LoginService
- NotaService
- ChamadaService
- AlunoService
- ConteudoService
- CursoService
- EventoService
- MateriaService
- ProfessorService
- TurmaService
- TokenService (JWT)
- UserService
- NotificacaoService
- BcryptPasswordHasher

**Models (15 total):**
- Aluno
- Professor
- Administrador
- Matricula
- Nota
- Chamada
- Curso
- Materia
- Turma
- Evento
- Conteudo
- Notificacao
- NotificacaoTurma
- ComprovantePagamento
- Documentos

### 2.2 Fluxo de Matrícula Completo

```
1. Candidato preenche: Nome, Email, CPF, Curso
   ↓
2. Backend valida CPF duplicado (mesmo curso)
   ↓
3. Cria Matricula com Status: AGUARDANDO_PAGAMENTO
   ↓
4. Candidato envia Comprovante de Pagamento
   Status → AGUARDANDO_DOCUMENTOS
   ↓
5. Candidato envia: Histórico + Cópia CPF
   Status → AGUARDANDO_APROVACAO
   ↓
6. Admin visualiza em /admin/solicitacoes
   ↓
7. Admin aprova:
   - Gera senha aleatória segura (12 chars)
   - Hash BCrypt
   - Cria Aluno vinculado à Matricula
   - Status → APROVADO
   - Envia evento para AWS SQS
   ↓
8. Serviço externo (Lambda/Worker) consome SQS
   Envia email com credenciais de acesso
   ↓
9. Novo aluno recebe email e faz login
   CPF + Senha temporária
   ↓
10. Aluno acessa dashboard
    Pode visualizar matérias, notas, eventos
```

### 2.3 Lógica de Notas

**Cálculo:**
```
Media = (Nota1 + Nota2) / 2

Status_Aprovacao:
  if QtdFaltas > 10:
    Status = "REPROVADO"
  else if Media >= 6:
    Status = "APROVADO"
  else:
    Status = "REPROVADO"

Nivel_Desempenho:
  if Media < 6:
    Nivel = "BRONZE"
  else if 6 <= Media <= 8:
    Nivel = "PRATA"
  else if Media > 8:
    Nivel = "OURO"
```

**Constraints:**
- UNIQUE(AlunoId, MateriaId) - um aluno tem uma nota por matéria
- Media é calculada automaticamente ao inserir

### 2.4 Controle de Frequência (Chamada)

**Regras:**
- Professor só pode fazer 1 chamada por matéria/turma/dia
- Se tentar 2x no mesmo dia → Exception
- Permite atualizar se erro no mesmo dia

**Fluxo:**
1. Professor acessa /professor/chamada
2. Seleciona Materia e Turma
3. Lista de alunos com checkboxes (Presente/Faltante)
4. POST /api/chamada/realizar ou /atualizar
5. Service verifica data/hora e turma
6. Cria ou atualiza registros

### 2.5 Endpoints da API

**Autenticação:**
```
POST /api/login
  Input: { Tipo, Cpf, Senha }
  Output: { id, nome, tipo, token, cursosIds, turmaId }
```

**Matrícula:**
```
POST /api/matricula - Criar (Public)
GET /api/matricula/pendentes - Listar (Admin/Prof)
POST /api/matricula/{id}/pagamento - Upload comprovante (Public)
POST /api/matricula/{id}/documentos - Upload docs (Public)
POST /api/matricula/aprovar/{id} - Aprovar (Admin)
POST /api/matricula/rejeitar/{id} - Rejeitar (Admin)
```

**Notas:**
```
POST /api/nota - Adicionar (Admin/Prof)
GET /api/nota/aluno/{alunoId} - Ver notas (Aluno)
GET /api/nota/desempenho/{alunoId} - Desempenho (Aluno)
GET /api/nota/relatorio/{alunoId} - PDF (Aluno)
```

**Chamada:**
```
POST /api/chamada/realizar - Primeira chamada do dia (Prof)
PUT /api/chamada/atualizar - Atualizar chamada (Prof)
```

**Outros Controllers:**
```
Turma, Materia, Curso, Professor, Aluno, Evento, Conteudo, Notificacao
```

### 2.6 Segurança Backend

**Autenticação JWT:**
- Algoritmo: HMAC SHA256
- Expiration: 8 horas (configurável)
- Claims: NameIdentifier, Name, Role, CPF
- Validação de Issuer e Audience

**Hash de Senha:**
- Algoritmo: BCrypt
- Work factor: Padrão (seguro vs brute-force)
- Sal: Aleatório por hash
- Irreversível: Impossível recuperar senha original

**Validação de Entrada:**
- CPF e Senha obrigatórios
- Tipo de usuário validado
- Tamanho de arquivo validado
- IDs positivos

**Exception Handling:**
- BusinessException (400)
- RequisicaoInvalidaException (400)
- CredenciaisInvalidasException (401)
- Generic Exception (500) com mensagem genérica

### 2.7 AWS SQS Integration

**Quando usado:**
1. Matrícula aprovada - enviar credenciais
2. Senha resetada - enviar link/senha temporária

**Classe SQSProducer:**
```csharp
public async Task EnviarEventoAsync<T>(T evento)
{
    var message = new SendMessageRequest
    {
        QueueUrl = _queueUrl,
        MessageBody = JsonConvert.SerializeObject(evento)
    };
    await _sqs.SendMessageAsync(message);
}
```

---

## 3. Frontend - React + TypeScript + Vite

### 3.1 Estrutura Completa

**Páginas:**
- LoginPage.tsx - Login de usuários
- MatriculaPage.tsx - Processo de matrícula
- aluno/DashboardAluno.tsx
- aluno/Calendario.tsx
- aluno/Desempenho.tsx
- aluno/Materias.tsx
- aluno/Relatorio.tsx
- aluno/MateriaDetalhes.tsx
- aluno/NotificacoesAluno.tsx
- professor/DashboardProfessor.tsx
- professor/CalendarioProfessor.tsx
- professor/Chamada.tsx
- professor/Boletins.tsx
- professor/Conteudo.tsx
- professor/Notas.tsx
- admin/DashboardAdmin.tsx
- admin/Solicitacoes.tsx
- admin/Usuarios.tsx
- admin/Cursos.tsx
- admin/Turmas.tsx
- admin/Professores.tsx
- admin/Notificacoes.tsx
- admin/Alunos.tsx

**Componentes Reutilizáveis:**
- Header.tsx - Banner com logo e tema
- Modal.tsx - Caixa de diálogo
- Loading.tsx - Spinner
- ErrorMessage.tsx - Mensagem de erro
- EmptyState.tsx - Estado vazio
- ThemeToggle.tsx - Dark/Light mode
- MateriaCard.tsx - Card de matéria
- CalendarioGrid.tsx - Grade de calendário
- EventoModal.tsx - Modal de eventos
- GraficoEvolucao.tsx - Gráfico com Recharts

**Layouts:**
- AlunoLayout.tsx
- ProfessorLayout.tsx
- AdminLayout.tsx
- DashboardLayout.tsx
- Sidebar.jsx

### 3.2 Context API - Autenticação Global

**AuthContext armazena:**
```typescript
interface AuthContextType {
  user: LoginResponse | null;
  token: string | null;
  isLoading: boolean;
  login: (credentials) => Promise<void>;
  logout: () => void;
}
```

**localStorage:**
- `usuarioLogado`: JSON com id, nome, tipo, cursosIds, turmaId
- `token`: JWT token

**PrivateRoute:**
- Valida autenticação antes de renderizar
- Redireciona para /login se não autenticado

### 3.3 Services (Integração API)

**api.ts - Configuração Axios:**
- Base URL: configurable via VITE_API_URL
- Timeout: 30 segundos
- Auto-adiciona Authorization header com JWT
- Trata 401 Unauthorized

**auth.service.ts:**
```typescript
export async function login(credentials: LoginCredentials)
export async function logout()
```

**aluno.service.ts:**
```typescript
export async function fetchMateriasAluno(cursoId: number)
export async function fetchDesempenhoAluno(alunoId: number)
export async function fetchAllNotas(alunoId: number)
```

### 3.4 Hooks Customizados

**useAluno.ts:**
```typescript
export function useMateriasAluno(cursoId: number | undefined)
export function useDesempenhoAluno(alunoId: number | undefined)
export function useNotasAluno(alunoId: number | undefined)
```

Usa React Query para:
- Cache automático
- Retry automático
- Enabled/Disabled baseado em dependencies

**useErrorHandler.ts:**
```typescript
export function useErrorHandler()
export function useNotification()
```

### 3.5 Schemas de Validação (Zod)

```typescript
// Login
export const LoginCredentialsSchema = z.object({
  Tipo: z.enum(['aluno', 'professor', 'administrador']),
  Cpf: z.string().min(11),
  Senha: z.string().min(1),
});

// Matrícula
export const EnrollmentDataSchema = z.object({
  nome: z.string().min(1),
  email: z.string().email(),
  cpf: z.string().min(11),
  cursoId: z.number(),
});
```

### 3.6 Rotas da Aplicação

**Públicas:**
- / → /matricula
- /login
- /matricula

**Aluno (Privadas):**
- /aluno (redirect para /dashboard)
- /aluno/dashboard
- /aluno/calendario
- /aluno/desempenho
- /aluno/materias
- /aluno/materias/:materiaId
- /aluno/relatorio
- /aluno/notificacoes

**Professor (Privadas):**
- /professor
- /professor/dashboardProfessor
- /professor/calendario
- /professor/boletim
- /professor/chamada
- /professor/conteudo
- /professor/notas

**Admin (Privadas):**
- /admin
- /admin/dashboard
- /admin/solicitacoes
- /admin/usuarios
- /admin/cursos
- /admin/turmas
- /admin/professores
- /admin/notificacoes
- /admin/alunos

---

## 4. Banco de Dados - SQL Server

### 4.1 Diagrama Entidade-Relacionamento

**Usuários:**
- Aluno (matriculaId FK, turmaId FK)
- Professor (professorId PK)
- Administrador

**Académico:**
- Curso (profResponsavel FK)
- Turma (cursoId FK)
- Materia (cursoId FK)
- Nota (alunoId FK, materiaId FK) - UNIQUE
- Chamada (matriculaId FK, materiaId FK, turmaId FK)
- Conteudo (materiaId FK, professorId FK, turmaId)

**Notificações:**
- Notificacao
- NotificacaoTurma (N:M junction)

**Matrícula:**
- Matricula (cursoId FK)
- ComprovantePagamento (matriculaId FK)
- Documentos (matriculaId FK)

**Eventos:**
- Evento

### 4.2 Campos Principais das Entidades

**Aluno:**
- Id (PK), Nome, Email, CPF, Senha (BCrypt), MatriculaId (FK), TurmaId (FK), Status

**Matricula:**
- Id (PK), Nome, Email, CPF, Status (AGUARDANDO_PAGAMENTO → APROVADO), CursoId (FK)

**Nota:**
- Id (PK), AlunoId (FK), MateriaId (FK), Nota1, Nota2, Media, QtdFaltas, Status
- Index: UNIQUE(AlunoId, MateriaId)

**Chamada:**
- Id (PK), MatriculaId (FK), MateriaId (FK), Faltou (bool), HorarioDaAula, TurmaId (FK)

### 4.3 Migrations

**Usar Entity Framework CLI:**
```bash
# Criar database
dotnet ef database create

# Ver status
dotnet ef migrations list

# Atualizar
dotnet ef database update

# Adicionar nova migration
dotnet ef migrations add [NomeMigracao]

# Reverter
dotnet ef database update [MigracaoAnterior]
```

---

## 5. Fluxos Detalhados

### 5.1 Login e Autenticação

```
Frontend → POST /api/login
         ↓
Backend → Valida CPF/Senha
         ↓
         → Busca Aluno/Professor/Admin
         ↓
         → Verifica BCrypt hash
         ↓
         → Se válido: Gera JWT
         ↓
         → Retorna: id, nome, tipo, token, cursosIds, turmaId
         ↓
Frontend → Armazena em localStorage
         ↓
         → Adiciona Authorization header
         ↓
         → Redireciona para /aluno ou /professor ou /admin
```

### 5.2 Desempenho do Aluno

```
Frontend → GET /api/nota/desempenho/{alunoId}
         ↓
Backend  → Service.GetDesempenhoByAlunoId
         ↓
         → Busca todas as Notas do aluno
         ↓
         → Para cada nota:
           - Extrai media (já calculada)
           - Extrai qtdFaltas
           - Calcula Nivel (BRONZE/PRATA/OURO)
         ↓
         → Retorna List<DesempenhoDTO>
         ↓
Frontend → Exibe tabela com cores por Nivel
         ↓
         → Renderiza gráfico com Recharts
```

### 5.3 Lançamento de Notas

```
Professor → Acessa /professor/notas
          ↓
          → Seleciona Materia e Turma
          ↓
Frontend  → GET /api/nota/turma/{id}/materia/{id}
          ↓
Backend   → Retorna lista de alunos
          ↓
Frontend  → Renderiza formulário com inputs de nota
          ↓
Professor → Preenche Nota1, Nota2 para cada aluno
          ↓
          → POST /api/nota (List<NotaDTORequest>)
          ↓
Backend   → Para cada nota:
            - Calcula Media = (N1 + N2) / 2
            - Obtém QtdFaltas
            - Calcula Status (APROVADO/REPROVADO)
            - Cria registro UNIQUE(AlunoId, MateriaId)
          ↓
          → Retorna sucesso
          ↓
Frontend  → Toast: "Notas lançadas com sucesso!"
```

### 5.4 Chamada (Frequência)

```
Professor → Acessa /professor/chamada
          ↓
          → Seleciona Materia e Turma
          ↓
Frontend  → GET /api/aluno/turma/{turmaId}
          ↓
Backend   → Lista alunos da turma
          ↓
Frontend  → Lista com checkboxes (Presente/Faltante)
          ↓
Professor → Marca presença/falta
          ↓
          → POST /api/chamada/realizar
             ou PUT /api/chamada/atualizar
          ↓
Backend   → Verifica se já existe chamada hoje
          ↓
          Se NÃO existe: Cria
          Se SIM e é atualizar: Atualiza
          Se SIM e é realizar: Exception
          ↓
Frontend  → Toast: "Chamada registrada!"
```

---

## 6. Segurança - Detalhamentos

### 6.1 Autenticação JWT

**Token contém:**
- NameIdentifier: ID do usuário
- Name: Nome do usuário
- Role: Tipo (aluno/professor/admin)
- CPF: CPF do usuário

**Validação:**
- Issuer: tivit-academy
- Audience: tivit-academy-users
- Expiration: 8 horas
- Signature: HMAC SHA256

### 6.2 Hashing de Senha

**BCrypt + Salt:**
- Nova senha gera novo hash
- Impossível reversão
- Seguro contra ataques

**Geração de Senha Aleatória:**
- 12 caracteres
- Inclui: letras, números, símbolos
- Criptograficamente segura

### 6.3 CORS

**Desenvolvimento:** AllowAnyOrigin()
**Produção:** Especificar domínios

### 6.4 Validação

**Input Validation:**
- CPF obrigatório
- Senha obrigatória
- Tipo validado contra enum
- Arquivo não pode ser nulo
- ID deve ser positivo

**Prevenção SQL Injection:**
- Entity Framework Core usa parameterized queries
- Nunca concatenar strings em queries

---

## 7. Instalação e Deployment

### 7.1 Backend

**Requisitos:**
- .NET 9 SDK
- SQL Server 2019+

**Passos:**
```bash
cd backend/tivitApi
dotnet restore
dotnet ef database update
dotnet run
```

**Configuração (appsettings.json):**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=TivitAcademy;Trusted_Connection=true;"
  },
  "Jwt": {
    "Key": "sua-chave-secreta-minimo-32-caracteres",
    "Issuer": "tivit-academy",
    "Audience": "tivit-academy-users",
    "ExpiresInHours": 8
  },
  "AWS": {
    "Region": "us-east-1",
    "SQSQueueUrl": "https://sqs.us-east-1.amazonaws.com/.../queue-name"
  }
}
```

### 7.2 Frontend

**Requisitos:**
- Node.js 18+
- npm ou yarn

**Passos:**
```bash
cd frontend
npm install
cp .env.example .env
# Editar VITE_API_URL=http://localhost:5027/api
npm run dev
```

**Build para Produção:**
```bash
npm run build
npm run preview
```

### 7.3 Docker

**Backend Dockerfile:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 as build
WORKDIR /app
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app/out .
EXPOSE 5027
ENTRYPOINT ["dotnet", "tivitApi.dll"]
```

---

## 8. Troubleshooting

| Problema | Solução |
|----------|---------|
| Bearer token invalid | Token expirou ou chave JWT incorreta |
| CORS error | Frontend URL não está em CORS permitidas |
| Database connection | SQL Server não está rodando ou connection string errada |
| 401 Unauthorized | Token inválido ou expirado |
| Migration failed | Reverter: `dotnet ef database update <anterior>` |
| Senha não funciona | Verificar hash BCrypt; Verifica senha correta |
| SQS error | Verificar credenciais AWS e URL da fila |

---

## 9. Tecnologias e Dependências Principais

**Backend:**
- .NET 9, ASP.NET Core
- Entity Framework Core 9 (SQL Server)
- AutoMapper
- BCrypt.Net-Next
- AWSSDK.SQS
- QuestPDF
- JWT Bearer Authentication

**Frontend:**
- React 18
- TypeScript 5.9
- Vite 7.3
- React Router 7
- Axios
- Zod (Validação)
- Recharts (Gráficos)
- React Hot Toast (Notificações)

---

## 10. Recursos e Documentação

- .NET: https://learn.microsoft.com/dotnet
- EF Core: https://learn.microsoft.com/ef
- React: https://react.dev
- TypeScript: https://www.typescriptlang.org
- Vite: https://vitejs.dev
- AWS SQS: https://docs.aws.amazon.com/sqs
- BCrypt: https://en.wikipedia.org/wiki/Bcrypt
- JWT: https://jwt.io

---

**Documentação Gerada em:** 18 de Março de 2026
**Versão do Projeto:** 1.0.0
**Status:** Projeto em Desenvolvimento Ativo
**Autor:** Documentação Técnica Automática
