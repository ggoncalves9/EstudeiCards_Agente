# EstudeiCards_Agente
O EstudeiCards Agentes é um sistema de gamificação de conhecimento DevOps

# 🧠 EstudeiCards Agentes

Projeto open-source de agente educacional com envio diário de perguntas sobre **Cloud e DevOps** via **WhatsApp**, utilizando **N8N + PostgreSQL + Zaia + LLMs (OpenAI/Gemini)**.

O objetivo é promover aprendizado contínuo e gamificado para membros da comunidade **DOUG BR**, com feedback imediato, pontuação semanal e ranking automático postado no LinkedIn.

## ✨ Funcionalidades

- Envio diário de 3 perguntas personalizadas com níveis (Fácil, Pleno, Expert)
- Correção automática com explicação (gabarito ou IA)
- Pontuação acumulada semanalmente
- Ranking Top 10 postado toda segunda-feira no LinkedIn
- Totalmente automatizado via N8N
- Uso de LLM para geração e avaliação dinâmica de perguntas
- Canal principal: WhatsApp via Agente Zaia

## 🧩 Stack Utilizada

- [x] **N8N** – Orquestração dos fluxos (Docker)
- [x] **PostgreSQL** – Armazenamento de usuários, perguntas, respostas e ranking
- [x] **Zaia** – Agente no WhatsApp (envio e recebimento das mensagens)
- [x] **OpenAI GPT-4 / Gemini** – Geração e correção dinâmica com LLM
- [x] **LinkedIn API** – Publicação do ranking semanal (apenas post público)

---

## ✅ Checklist de Implementação

### 📁 Infraestrutura

- [ ] Subir ambiente local de desenvolvimento (Docker Compose com PostgreSQL, N8N)
- [ ] Criar banco de dados com as tabelas: `usuarios`, `perguntas`, `respostas`, `pontuacoes`, `ranking`
- [ ] Criar script inicial de seed para banco de perguntas (por nível)

### 🤖 Agente Zaia

- [ ] Criar agente na plataforma Zaia com integração ao WhatsApp
- [ ] Configurar steps:
  - [ ] Receber mensagens dos usuários
  - [ ] Enviar mensagens via API do N8N
  - [ ] Capturar resposta e enviar ao Webhook do N8N
  - [ ] Retornar feedback formatado
- [ ] Testar integração Zaia + N8N com mensagem de boas-vindas

### 🔄 Fluxos N8N

- [ ] Criar fluxo de **envio diário de perguntas** (cron → PostgreSQL → Zaia)
- [ ] Criar fluxo de **recepção e correção de resposta** (webhook → DB/LLM → feedback)
- [ ] Criar fluxo de **cálculo e reset semanal** com postagem no LinkedIn
- [ ] Criar fallback para inatividade ou resposta incompleta

### 🧠 LLM / Correção

- [ ] Conectar à API do OpenAI (ou Gemini)
- [ ] Criar prompt base para avaliação de resposta + geração de explicação
- [ ] Integrar ao fluxo de correção no N8N (com fallback para gabarito fixo)

### 📣 Divulgação e Ranking

- [ ] Criar post automático via API do LinkedIn com Top 10 semanal
- [ ] Testar com template de postagem (nome, pontuação, emojis)
- [ ] Validar permissão de postagem na página DOUG BR

---

## 💡 Futuras melhorias

- [ ] Geração de certificados/badges semanais (imagem)
- [ ] Dashboard web para visualizar pontuação e histórico
- [ ] Tradução multiidioma (pt/en)

---

## 📌 Licença

Este projeto é open-source e gratuito, feito para fortalecer o aprendizado coletivo da comunidade **DevOps User Group Brazil**.

---

> **Desenvolvido por:** Guilherme Gonçalves – DevOps & Automação Educacional  
> 💬 Para dúvidas ou sugestões: [LinkedIn](https://www.linkedin.com/in/ggoncalves9) | [Instagram](https://www.instagram.com/realmirage.ia/) | [Empresa Site](https://realmirage.com.br/)


