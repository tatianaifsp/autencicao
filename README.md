
# Para criar uma aplicação de autenticação com Vue CLI, Node.js, MongoDB, JWT, dotenv, CORS e Insomnia, vou guiar você em um passo a passo com configuração do back-end e front-end.

O projeto será estruturado em:
Configuração do back-end com Node.js, Express, MongoDB e JWT.
Configuração do front-end com Vue CLI e Axios.
Configuração do Insomnia para testar as rotas de API.

# ________________________Parte 1: Back-end com Node.js, Express, MongoDB e JWT

# 1. Configuração Inicial do Projeto Back-end
Crie uma pasta para o projeto do back-end e instale as dependências:

Copiar código
mkdir backend-auth && cd backend-auth
npm init -y
npm install express mongoose bcryptjs dotenv jsonwebtoken cors

Estrutura de Arquivos do Back-end
Estruture o projeto da seguinte forma:

backend-auth/
├── config/
│   └── db.js
├── controllers/
│   └── authController.js
├── middlewares/
│   └── authMiddleware.js
├── models/
│   └── User.js
├── routes/
│   └── authRoutes.js
├── .env
├── server.js
└── package.json
Configuração do .env

# Crie o arquivo .env e adicione as variáveis de ambiente:
env
      Copiar código
PORT=5000
MONGO_URI=sua_uri_mongodb
JWT_SECRET=sua_chave_secreta

# 2. Conexão com MongoDB (config/db.js)
Copiar código

const mongoose = require("mongoose");
const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    console.log("MongoDB conectado!");
  } catch (error) {
    console.error("Erro de conexão:", error);
    process.exit(1);
  }
};

module.exports = connectDB;

# 3. Modelo de Usuário (models/User.js)
Copiar código
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
});

userSchema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
  next();
});

module.exports = mongoose.model("User", userSchema);

# 4. Controlador de Autenticação (controllers/authController.js)
Copiar código

const User = require("../models/User");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");

exports.register = async (req, res) => {
  try {
    const { name, email, password } = req.body;
    const user = new User({ name, email, password });
    await user.save();
    res.status(201).json({ message: "Usuário registrado!" });
  } catch (error) {
    res.status(500).json({ message: "Erro no registro", error });
  }
};

exports.login = async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(400).json({ message: "Credenciais inválidas" });
  }
  const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET, { expiresIn: "1h" });
  res.json({ token });
};

# 5. Middleware de Autenticação (middlewares/authMiddleware.js)

Copiar código
const jwt = require("jsonwebtoken");

const authMiddleware = (req, res, next) => {
  const token = req.headers["authorization"];
  if (!token) return res.status(401).json({ message: "Token não fornecido" });

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "Token inválido" });
    req.userId = decoded.userId;
    next();
  });
};

module.exports = authMiddleware;

# 6. Rotas de Autenticação (routes/authRoutes.js)
Copiar código

const express = require("express");
const router = express.Router();
const authController = require("../controllers/authController");

router.post("/register", authController.register);
router.post("/login", authController.login);

module.exports = router;

# 7. Inicialização do Servidor (server.js)
Copiar código

require("dotenv").config();
const express = require("express");
const cors = require("cors");
const connectDB = require("./config/db");
const authRoutes = require("./routes/authRoutes");

connectDB();
const app = express();
app.use(cors());
app.use(express.json());
app.use("/api/auth", authRoutes);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));


# _____________________________________________ Parte 2: Front-end com Vue CLI e Axios
1. Criando o Projeto Vue com Vue CLI
Crie um projeto Vue com Vue CLI:

Copiar código

vue create frontend-auth
cd frontend-auth
npm install axios

# Configuração do .env no Front-end
Crie o arquivo .env no diretório do front-end com a URL da API:
env
Copiar código

VUE_APP_API_URL=http://localhost:5000/api/auth

# 2. Serviço de Autenticação (src/services/authService.js)
Crie o arquivo authService.js na pasta src/services:
Copiar código

import axios from "axios";
// URL base da API, definida no .env
const API_URL = process.env.VUE_APP_API_URL;

// Função para registrar um novo usuário
export const register = async (userData) => {
  return await axios.post(`${API_URL}/register`, userData);
};

// Função para login de usuário
export const login = async (userData) => {
  const response = await axios.post(`${API_URL}/login`, userData);
  if (response.data.token) {
    localStorage.setItem("token", response.data.token);
  }
  return response.data;
};

// Função para logout, removendo o token do localStorage
export const logout = () => {
  localStorage.removeItem("token");
};

// Função para obter o token armazenado no localStorage
export const getToken = () => {
  return localStorage.getItem("token");
};

# 3. Configuração de Rotas e Componentes de Autenticação
No arquivo src/router/index.js, defina as rotas para as páginas de Registro e Login:
src/router/index.js

Copiar código

import { createRouter, createWebHistory } from "vue-router";
import Register from "../views/Register.vue";
import Login from "../views/Login.vue";

const routes = [
  { path: "/register", component: Register },
  { path: "/login", component: Login },
];

const router = createRouter({
  history: createWebHistory(),
  routes,
});

export default router;

# Componente de Registro (src/views/Register.vue)
Copiar código

<template>
  <div>
    <h2>Registro</h2>
    <form @submit.prevent="registerUser">
      <input v-model="name" placeholder="Nome" required />
      <input v-model="email" placeholder="Email" required />
      <input v-model="password" type="password" placeholder="Senha" required />
      <button type="submit">Registrar</button>
    </form>
  </div>
</template>

<script>
import { register } from "../services/authService";

export default {
  data() {
    return { name: "", email: "", password: "" };
  },
  methods: {
    async registerUser() {
      try {
        await register({
          name: this.name,
          email: this.email,
          password: this.password,
        });
        alert("Registrado com sucesso!");
        this.$router.push("/login");
      } catch (error) {
        console.error("Erro ao registrar:", error);
        alert("Erro no registro. Tente novamente.");
      }
    },
  },
};
</script>

# Componente de Login (src/views/Login.vue)
Copiar código

<template>
  <div>
    <h2>Login</h2>
    <form @submit.prevent="loginUser">
      <input v-model="email" placeholder="Email" required />
      <input v-model="password" type="password" placeholder="Senha" required />
      <button type="submit">Entrar</button>
    </form>
  </div>
</template>

<script>
import { login } from "../services/authService";

export default {
  data() {
    return { email: "", password: "" };
  },
  methods: {
    async loginUser() {
      try {
        await login({
          email: this.email,
          password: this.password,
        });
        alert("Login bem-sucedido!");
        this.$router.push("/");
      } catch (error) {
        console.error("Erro ao fazer login:", error);
        alert("Credenciais inválidas. Tente novamente.");
      }
    },
  },
};
</script>

# Parte 3: Testando a API com Insomnia
Registro de Usuário

Configure uma requisição POST para http://localhost:5000/api/auth/register.
Envie um corpo JSON com name, email e password.
Login de Usuário

Configure uma requisição POST para http://localhost:5000/api/auth/login.
Envie um corpo JSON com email e password do usuário.
Verifique se o token JWT é retornado na resposta.

# Para iniciar o projeto Vue, vá para o diretório do front-end e execute:
Copiar código
npm run serve

#### Isso conclui o front-end em Vue CLI integrado ao back-end com Node.js e MongoDB.

****************************************************************************************************************

Para implementar uma navegação que redireciona o usuário autenticado para uma página de **Dashboard** após o login e que, ao registrar, ele seja automaticamente logado, vamos configurar o front-end e o back-end para trabalhar com o MongoDB e gerenciar o fluxo de autenticação e navegação entre as páginas.

Aqui está o passo a passo para criar esse fluxo:

*******************************************************************************************************************

### Parte 1: Configurando o Back-end com MongoDB, Express, e JWT

#### Estrutura de Dados no MongoDB

Usaremos uma coleção chamada `User` para armazenar os dados dos usuários. O esquema de dados para um usuário incluirá `name`, `email`, e `password`.

1. **Modelo do Usuário (models/User.js)**

   - Esse modelo define o esquema de dados do usuário e configura a criptografia da senha.

   const mongoose = require("mongoose");
   const bcrypt = require("bcryptjs");

   const userSchema = new mongoose.Schema({
     name: { type: String, required: true },
     email: { type: String, required: true, unique: true },
     password: { type: String, required: true },
   });

   // Middleware para criptografar a senha antes de salvar
   userSchema.pre("save", async function (next) {
     if (!this.isModified("password")) return next();
     const salt = await bcrypt.genSalt(10);
     this.password = await bcrypt.hash(this.password, salt);
     next();
   });

   module.exports = mongoose.model("User", userSchema);

2. **Controlador de Autenticação (controllers/authController.js)**

   - Esse controlador gerencia o registro, o login e a verificação de credenciais.
     
   const User = require("../models/User");
   const jwt = require("jsonwebtoken");
   const bcrypt = require("bcryptjs");

   // Função de registro
   exports.register = async (req, res) => {
     try {
       const { name, email, password } = req.body;
       const user = new User({ name, email, password });
       await user.save();
       res.status(201).json({ message: "Usuário registrado com sucesso!" });
     } catch (error) {
       res.status(500).json({ message: "Erro no registro", error });
     }
   };

   // Função de login
   exports.login = async (req, res) => {
     const { email, password } = req.body;
     const user = await User.findOne({ email });
     if (!user || !(await bcrypt.compare(password, user.password))) {
       return res.status(400).json({ message: "Credenciais inválidas" });
     }
     const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET, { expiresIn: "1h" });
     res.json({ token });
   };

3. **Rotas de Autenticação (routes/authRoutes.js)**

   - Definimos as rotas para **Registro** e **Login**.

   const express = require("express");
   const router = express.Router();
   const authController = require("../controllers/authController");

   router.post("/register", authController.register);
   router.post("/login", authController.login);

   module.exports = router;

### Parte 2: Front-end com Vue CLI e Axios para Registro, Login, e Dashboard

1. **Configuração do Serviço de Autenticação (`src/services/authService.js`)**

   - Este serviço contém as funções para **registrar**, **logar** e **obter** o token JWT.


   import axios from "axios";
   const API_URL = process.env.VUE_APP_API_URL;

   // Função para registrar um novo usuário
   export const register = async (userData) => {
     return await axios.post(`${API_URL}/register`, userData);
   };

   // Função para login de usuário
   export const login = async (userData) => {
     const response = await axios.post(`${API_URL}/login`, userData);
     if (response.data.token) {
       localStorage.setItem("token", response.data.token); // Armazena o token no localStorage
     }
     return response.data;
   };

   // Função para obter o token armazenado no localStorage
   export const getToken = () => {
     return localStorage.getItem("token");
   };
   ```

2. **Tela de Autenticação (src/views/Auth.vue)**

   - Este componente renderiza a interface para **Login** e **Registro** e redireciona o usuário para o **Dashboard** após a autenticação.

   <template>
     <div class="auth-container">
       <h2 v-if="isLoginMode">Login</h2>
       <h2 v-else>Registrar</h2>

       <form @submit.prevent="handleAuth">
         <input v-if="!isLoginMode" v-model="name" placeholder="Nome" required />
         <input v-model="email" placeholder="Email" required />
         <input v-model="password" type="password" placeholder="Senha" required />
         <button type="submit">{{ isLoginMode ? 'Entrar' : 'Registrar' }}</button>
       </form>

       <p @click="toggleMode">
         {{ isLoginMode ? 'Não tem conta? Registrar' : 'Já tem conta? Fazer login' }}
       </p>
     </div>
   </template>

   <script>
   import { login, register } from "../services/authService";

   export default {
     data() {
       return {
         isLoginMode: true,
         name: "",
         email: "",
         password: "",
       };
     },
     methods: {
       toggleMode() {
         this.isLoginMode = !this.isLoginMode;
         this.clearFields();
       },
       async handleAuth() {
         try {
           if (this.isLoginMode) {
             await login({ email: this.email, password: this.password });
             this.$router.push("/dashboard"); // Redireciona para Dashboard
           } else {
             await register({ name: this.name, email: this.email, password: this.password });
             await login({ email: this.email, password: this.password });
             this.$router.push("/dashboard"); // Redireciona após registro e login
           }
         } catch (error) {
           console.error("Erro:", error);
           alert("Erro na autenticação.");
         }
       },
       clearFields() {
         this.name = "";
         this.email = "";
         this.password = "";
       },
     },
   };
   </script>
 

3. **Dashboard (src/views/Dashboard.vue)**

   - Página simples que representa o **Dashboard** após o login.

   <template>
     <div>
       <h1>Bem-vindo ao Dashboard</h1>
       <p>Você está autenticado!</p>
     </div>
   </template>

   <script>
   export default {
     mounted() {
       // Verifica se há um token armazenado
       if (!localStorage.getItem("token")) {
         this.$router.push("/auth"); // Redireciona para login se não estiver autenticado
       }
     },
   };
   </script>
   ```

4. **Configuração de Rotas (src/router/index.js)**

   - Adicione as rotas para `Auth` e `Dashboard`.

   import { createRouter, createWebHistory } from "vue-router";
   import Auth from "../views/Auth.vue";
   import Dashboard from "../views/Dashboard.vue";

   const routes = [
     { path: "/auth", component: Auth },
     { path: "/dashboard", component: Dashboard },
     { path: "/", redirect: "/auth" },
   ];

   const router = createRouter({
     history: createWebHistory(),
     routes,
   });

   export default router;

### Parte 3: Testando com Insomnia e Salvando no MongoDB

1. **Registro de Usuário**
   - Envie uma requisição `POST` para `http://localhost:5000/api/auth/register` com `name`, `email`, e `password`.
   - Verifique se o usuário é salvo na coleção `User` no MongoDB.

2. **Login de Usuário**
   - Envie uma requisição `POST` para `http://localhost:5000/api/auth/login` com `email` e `password`.
   - Verifique se o token JWT é retornado na resposta.

### Conclusão
Essa configuração completa o fluxo de autenticação, permitindo que o usuário registre-se, faça login e acesse o **Dashboard**. O MongoDB armazena todos os usuários na coleção `User`, e o token JWT garante a autenticação do usuário para acessar a página do **Dashboard** após o login.





  
