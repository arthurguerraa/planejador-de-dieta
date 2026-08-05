# Especificação do Projeto — App de Montagem de Dieta

## 1. Visão Geral

Aplicação full stack onde o usuário se cadastra, monta suas próprias dietas escolhendo alimentos de uma base de dados já cadastrada, organiza os alimentos em refeições, e acompanha em tempo real os totais de calorias, carboidratos, proteínas e gorduras da dieta montada, comparando com uma meta calórica diária definida pelo próprio usuário.

## 2. Stack Tecnológica

- **Frontend:** React (com Vite)
- **Backend:** Node.js + Express
- **Banco de dados:** MySQL
- **Acesso ao banco:** SQL puro com a biblioteca `mysql2` (sem ORM)
- **Autenticação:** JWT implementado do zero (bcrypt para hash de senha, geração/validação manual de token)

## 3. Base de Dados de Alimentos

- Utilizar a **TACO (Tabela Brasileira de Composição de Alimentos)** — UNICAMP — como fonte dos dados nutricionais.
- Os dados da TACO são importados **uma única vez** via script de seed para o banco MySQL próprio da aplicação (não é uma dependência de API externa em runtime).
- Categorias de alimentos: usar os grupos alimentares já definidos pela própria TACO (ex.: Cereais e derivados, Verduras e hortaliças, Frutas e derivados, Carnes e derivados, Leite e derivados, Leguminosas, Ovos, Gorduras e óleos, Pescados, Bebidas, Produtos açucarados, Nozes e sementes, Alimentos preparados, Outros/Miscelânea). Essas categorias alimentam o filtro de busca no frontend.

## 4. Modelo de Dados (MySQL)

```sql
users
 id (PK), name, email (unique), password_hash, daily_goal_kcal, created_at

foods                          -- populada via seed com dados da TACO
 id (PK), name, category, kcal_100g, carbs_100g, protein_100g, fat_100g

food_portions                  -- medidas caseiras (1:N com foods)
 id (PK), food_id (FK -> foods.id), description ("1 fatia", "1 unidade"), grams_equivalent

diets
 id (PK), user_id (FK -> users.id), name, created_at

meals
 id (PK), diet_id (FK -> diets.id), name, order

diet_items
 id (PK), meal_id (FK -> meals.id), food_id (FK -> foods.id), quantity_grams
```

### Regras de negócio do modelo

- `diet_items.quantity_grams` **sempre armazena a quantidade em gramas**, mesmo que o usuário tenha selecionado uma porção (medida caseira) na interface — a conversão porção → gramas acontece antes de salvar.
- Os totais de macros (por refeição e por dieta) **nunca são armazenados** em tabela — são sempre calculados dinamicamente a partir de `foods` + `diet_items`, evitando inconsistência caso um alimento seja atualizado no futuro.
- Um usuário pode ter **múltiplas dietas** (relação 1:N entre `users` e `diets`), permitindo, por exemplo, "Dieta de Cutting", "Dieta de Bulking", etc., sem precisar sobrescrever uma dieta existente.

## 5. Fluxo de Telas

```
/ (Landing Page)
 ├── /register (Cadastro)
 ├── /login (Login)
 └── (após autenticação)
      ├── /dietas (Minhas Dietas — lista de dietas do usuário)
      ├── /dietas/:id (Editor de Dieta)
      │    └── modal "Adicionar alimento" (busca por nome + filtro por categoria + escolha de porção ou gramas)
      └── /perfil (Perfil — meta calórica diária, dados do usuário)
```

### 5.1 Landing Page (`/`)
Página pública de apresentação do produto, sem chamadas à API (exceto navegação):
- **Hero**: título de impacto + botões "Cadastre-se" e "Já tenho conta"
- **Como funciona**: 3 passos (escolher alimentos → montar refeições → acompanhar macros)
- **Destaques**: base TACO, cálculo automático, múltiplas dietas, meta calórica personalizada
- **Rodapé**: links institucionais/portfólio

### 5.2 Cadastro / Login
Formulários simples de autenticação (nome, email, senha no cadastro; email e senha no login).

### 5.3 Minhas Dietas (`/dietas`)
- Lista de dietas do usuário logado, cada card mostrando nome + resumo rápido de macros (ex.: "Dieta de Cutting — 1850 kcal")
- Botão para criar nova dieta
- Clique em uma dieta → abre o Editor de Dieta

### 5.4 Editor de Dieta (`/dietas/:id`)
- Nome da dieta (editável)
- Lista de refeições (usuário pode adicionar, renomear e remover refeições — ex.: Café da manhã, Almoço, Jantar, Lanche)
- Dentro de cada refeição: lista de alimentos adicionados + botão "Adicionar alimento"
- **Painel de resumo de macros** (atualizado em tempo real):
  - Meta calórica diária (definida pelo usuário)
  - Totais consumidos: kcal, carboidratos (g), proteínas (g), gorduras (g)
  - Diferença entre meta e total consumido ("faltam X kcal" / "excedeu em X kcal")
  - Subtotal de macros por refeição, além do total geral da dieta

### 5.5 Modal "Adicionar Alimento"
- Busca por nome + filtro por categoria (grupos da TACO)
- Ao selecionar um alimento: input de quantidade em gramas OU seleção de uma porção/medida caseira pré-cadastrada
- Confirmação adiciona o item à refeição escolhida

### 5.6 Perfil (`/perfil`)
- Dados do usuário (nome, email)
- Meta calórica diária (editável)

## 6. Endpoints da API

### Auth (`/api/auth`) — públicas
```
POST   /api/auth/register       → cria usuário (name, email, password)
POST   /api/auth/login          → valida credenciais, retorna JWT
```

### Usuário (`/api/users`) — protegidas (JWT)
```
GET    /api/users/me            → dados do usuário logado
PUT    /api/users/me            → atualiza nome / meta calórica diária
```

### Alimentos (`/api/foods`) — protegidas (JWT)
```
GET    /api/foods               → lista alimentos (query params: ?search=&category=)
GET    /api/foods/:id           → detalhes de um alimento
GET    /api/foods/:id/portions  → medidas caseiras daquele alimento
GET    /api/foods/categories    → lista de categorias (para filtro no frontend)
```

### Dietas (`/api/diets`) — protegidas (JWT)
```
GET    /api/diets               → lista dietas do usuário logado (com resumo de macros)
POST   /api/diets               → cria nova dieta (name)
GET    /api/diets/:id           → detalhes da dieta (refeições + itens + totais calculados)
PUT    /api/diets/:id           → edita nome da dieta
DELETE /api/diets/:id           → remove dieta
```

### Refeições (`/api/diets/:dietId/meals` e `/api/meals`) — protegidas (JWT)
```
POST   /api/diets/:dietId/meals → adiciona refeição à dieta (name, order)
PUT    /api/meals/:id           → edita refeição
DELETE /api/meals/:id           → remove refeição
```

### Itens da dieta (`/api/meals/:mealId/items` e `/api/diet-items`) — protegidas (JWT)
```
POST   /api/meals/:mealId/items → adiciona alimento à refeição (food_id, quantity_grams)
PUT    /api/diet-items/:id      → edita quantidade do item
DELETE /api/diet-items/:id      → remove item da refeição
```

### 6.1 Regra de cálculo de macros

O cálculo dos macros é feito **inteiramente no backend**, dentro do endpoint `GET /api/diets/:id`. O frontend apenas exibe os valores já calculados, sem duplicar lógica de soma/conversão em JavaScript. Exemplo de resposta:

```json
{
  "id": 1,
  "name": "Dieta de Cutting",
  "daily_goal_kcal": 2200,
  "meals": [
    {
      "id": 1,
      "name": "Café da manhã",
      "items": [
        {
          "id": 1,
          "food_name": "Arroz branco cozido",
          "quantity_grams": 150,
          "kcal": 195,
          "carbs": 42.9,
          "protein": 3.9,
          "fat": 0.3
        }
      ],
      "meal_totals": { "kcal": 195, "carbs": 42.9, "protein": 3.9, "fat": 0.3 }
    }
  ],
  "diet_totals": { "kcal": 195, "carbs": 42.9, "protein": 3.9, "fat": 0.3 }
}
```

## 7. Estrutura de Pastas

### 7.1 Backend (`/backend`)

```
backend/
├── src/
│   ├── config/
│   │   └── database.js          # conexão com MySQL (pool mysql2)
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── userController.js
│   │   ├── foodController.js
│   │   ├── dietController.js
│   │   ├── mealController.js
│   │   └── dietItemController.js
│   ├── models/                  # funções de acesso ao banco (queries SQL)
│   │   ├── userModel.js
│   │   ├── foodModel.js
│   │   ├── dietModel.js
│   │   ├── mealModel.js
│   │   └── dietItemModel.js
│   ├── routes/
│   │   ├── authRoutes.js
│   │   ├── userRoutes.js
│   │   ├── foodRoutes.js
│   │   ├── dietRoutes.js
│   │   ├── mealRoutes.js
│   │   └── dietItemRoutes.js
│   ├── middlewares/
│   │   ├── authMiddleware.js     # valida JWT
│   │   └── errorHandler.js
│   ├── utils/
│   │   └── macroCalculator.js    # cálculo de macros (gramas → kcal/carb/prot/gord)
│   ├── database/
│   │   ├── schema.sql            # script de criação das tabelas
│   │   └── seed/
│   │       ├── taco_import.js    # script que lê a TACO e popula a tabela foods
│   │       └── taco_data.csv     # dados brutos da TACO
│   └── app.js                    # configuração do Express
├── server.js                     # entrada da aplicação
├── .env                          # DB_HOST, DB_USER, DB_PASSWORD, JWT_SECRET, etc.
└── package.json
```

### 7.2 Frontend (`/frontend`)

```
frontend/
├── src/
│   ├── assets/                   # imagens, ícones
│   ├── components/
│   │   ├── layout/                # Navbar, Footer, PrivateRoute
│   │   ├── diet/                  # MacroSummary, MealCard, DietItemRow
│   │   └── food/                  # FoodSearchModal, FoodCard
│   ├── pages/
│   │   ├── LandingPage.jsx
│   │   ├── LoginPage.jsx
│   │   ├── RegisterPage.jsx
│   │   ├── DietsListPage.jsx     # "Minhas Dietas"
│   │   ├── DietEditorPage.jsx    # editor com refeições + resumo de macros
│   │   └── ProfilePage.jsx
│   ├── services/                 # chamadas à API
│   │   ├── api.js                # instância base (baseURL, interceptor de JWT)
│   │   ├── authService.js
│   │   ├── foodService.js
│   │   └── dietService.js
│   ├── context/
│   │   └── AuthContext.jsx       # estado global do usuário logado
│   ├── hooks/
│   │   └── useAuth.js
│   ├── routes/
│   │   └── AppRoutes.jsx         # definição das rotas (react-router-dom)
│   ├── App.jsx
│   └── main.jsx
├── .env                          # VITE_API_URL
└── package.json
```

## 8. Resumo das Decisões Tomadas

| Decisão | Escolha |
|---|---|
| Fonte de dados de alimentos | TACO (importada via seed para o MySQL) |
| Unidade de quantidade | Gramas (livre) ou porção/medida caseira (convertida para gramas ao salvar) |
| Categorização de alimentos | Grupos alimentares oficiais da TACO |
| Acesso ao banco | SQL puro (`mysql2`), sem ORM |
| Escopo de dietas | Múltiplas dietas por usuário |
| Organização da dieta | Dividida em refeições, com visão geral e por refeição |
| Meta calórica | Definida pelo usuário no perfil, exibida no resumo da dieta |
| Autenticação | JWT implementado do zero (bcrypt + geração/validação manual de token) |
| Local do cálculo de macros | Backend (endpoint retorna valores já calculados) |
