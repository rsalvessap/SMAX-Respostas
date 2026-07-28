# SMAX Respostas - TJSP

Ferramenta de automacao e atendimento em lote para o sistema SMAX do Tribunal de Justica de Sao Paulo.

**Versao atual:** 1.7

---

## 1. Pre-requisito: instalar o Tampermonkey

1. Instale a extensao **Tampermonkey** na loja do seu navegador (Chrome, Edge ou Firefox)
2. Va em **Gerenciar Extensoes** e ative o **Modo do desenvolvedor**
3. No Tampermonkey, acesse **Painel de Controle** > **Configuracoes** e marque:
   - Permitir scripts de usuario
   - Permitir acesso a abas
   - Permitir requisicoes remotas

---

## 2. Instalacao

Clique no link abaixo para instalar diretamente pelo Tampermonkey:

**[Instalar SMAX Respostas - TJSP](https://github.com/rsalvessap/SMAX-Respostas/raw/refs/heads/master/SMAX/SMAX%20Respostas%20-%20TJSP.user.js)**

> O Tampermonkey abrira uma aba de confirmacao. Clique em **Instalar**.

O script funciona no SMAX e no eProc -- nao e necessario instalar nada separado.

---

## 3. Configuracao inicial

Ao abrir o SMAX na tela de chamados, uma **engrenagem** aparecera no canto inferior direito. Clique nela para abrir o painel de configuracoes.

### 3.1 Identificacao

No campo **"Quem e voce?"** (aba Geral), busque e selecione seu proprio nome. Isso vincula suas acoes ao seu usuario no sistema.

### 3.2 Configuracao compartilhada (SharedConfig)

Na aba **Geral**, na secao **Config. Compartilhada**, voce vera a URL do SharedConfig ja preenchida por padrao:

```
https://raw.githubusercontent.com/rsalvessap/SMAX-TOOLS/master/shared-config.json
```

Esta URL aponta para um arquivo JSON hospedado no GitHub que contem:
- Equipes e regras de roteamento
- Mapeamento de nomes e digitos
- Lista de ausentes
- Scripts compartilhados de solucao e discussao

**Nao altere esta URL** a menos que seja orientado a faze-lo. A sincronizacao e automatica: o script busca atualizacoes a cada 1 hora e aplica as configuracoes recebidas.

Para forcar uma atualizacao manual, clique no botao **Atualizar** ao lado da URL.

### 3.3 Equipes

As equipes sao configuradas e gerenciadas centralmente pelo administrador. Voce recebe automaticamente:

- Definicoes de equipes (2.2.1, 2.2.2, Geral, etc.)
- Regras de roteamento por GSE, local de divulgacao e texto
- Membros e faixas de digitos de cada equipe
- Assinaturas de equipe

Equipes compartilhadas aparecem com o badge **Compartilhada** e sao somente leitura.

### 3.4 Mensagem de recebimento

Na aba Geral, em **Mensagem de Recebimento**, voce pode personalizar o texto que sera enviado como discussao publica ao clicar em "Recebimento". Cada linha do texto vira um paragrafo no envio.

---

## 4. Modulos

### 4.1 ResponseHUD (Painel de Respostas)

Acesse via **Configuracoes > Respostas > Abrir ResponseHUD**.

Painel completo para atendimento e respostas em lote.

#### Painel esquerdo -- Filtros e lista

- **Filtro por equipes** -- pills com indicador azul (GSE na API) ou amarelo (regex local)
- **Filtro por Status Operacional** -- pills gerados automaticamente
- **Filtro por Status** -- Em Andamento, Pronto, Pendente, etc.
- **Filtro por Especialista** -- mostra apenas chamados de um atendente especifico
- **Filtro de texto** -- busca em tempo real por descricao, solicitante ou localizacao
- **Presets salvos** -- salve combinacoes de filtros para reutilizar. Preset fixo "Rejeitados" sempre disponivel
- **Ordenacao** -- por ID, data, status ou especialista (ascendente/descendente)
- **Busca por numero** -- localiza qualquer chamado pelo ID, mesmo fora dos filtros
- **Selecao em lote** -- checkbox "Selecionar todos" para operacoes em massa
- **Scroll virtual** -- renderiza apenas itens visiveis para suportar listas grandes

#### Painel direito -- Detalhe e acoes

**Cabecalho:**
- ID do chamado (link para o SMAX), badges VIP e Global, solicitante, localizacao, numero do processo CNJ, data de criacao

**Barra de acoes (chips):**

| Chip | Funcao |
|------|--------|
| GSE | Alterar grupo de suporte. Opcao "Com encaminhamento" posta discussao com texto configuravel |
| Especialista | Designar atendente. Busca por nome |
| Status | Alterar status do chamado (16 opcoes) |
| Status Operacional | Alterar situacao operacional (35+ opcoes TJSP) |
| Seguidor | Adicionar/remover seguidores do chamado |
| Seguir | Adicionar a si mesmo como seguidor |
| Recebimento | Postar discussao publica para o solicitante (texto editavel nas configuracoes) |
| Escalar | Transicao do chamado de Validacao para Atendimento |

**Editor de solucao:**
- Editor rico completo: negrito, italico, sublinhado, tachado, listas, links, cores, tamanhos de fonte, limpeza de formatacao
- Seletor de assinaturas (equipe e pessoais)
- Seletor de scripts de solucao (locais + compartilhados)
- Codigo de finalizacao: Atendido Offline, Suporte ao Vivo, Incidente Resolvido

**Painel de discussoes:**
- Lista de todas as discussoes do chamado com badges de privacidade (PUBLIC/INTERNAL)
- Botao "Replicar" copia conteudo para o editor de solucao
- Modal de expansao com navegacao entre discussoes
- **Nova discussao:** editor proprio com seletor de destinatario (Agente/Usuario/Fornecedor) e objetivo (Atualizacao/Acompanhamento/Resolucao/etc.)
- Seletor de scripts de discussao com preenchimento automatico de destinatario e objetivo

**Anexos:**
- Chips de anexos do chamado. Imagens abrem em modal com navegacao por teclado. PDFs abrem em nova aba.

**Envio (ENVIAR):**
- Chamado unico: executa diretamente
- Lote: exibe painel de confirmacao com tabela detalhada por chamado
- Grava: solucao, GSE, especialista, status, status operacional, escalacao, encaminhamento, recebimento, seguidores

### 4.2 Scripts/Templates

Acesse via **Configuracoes > Scripts**.

- **Duas abas:** Solucao e Discussao
- Crie, edite e exclua scripts localmente
- Scripts de discussao aceitam campos opcionais: "Para" e "Objetivo"
- **Sincronizacao:** importa scripts do Gerenciador de Chamados (Supabase) ou do SharedConfig (GitHub)
- **Exportar/Importar:** backup e restauracao em JSON
- Scripts compartilhados aparecem com badge "Compartilhado" e nao podem ser editados localmente

### 4.3 Assinaturas

Acesse via **Configuracoes > Assinaturas**.

- **Assinaturas pessoais:** criar, editar e excluir com pre-visualizacao em tempo real
- **Assinaturas por equipe:** configuradas centralmente pelo administrador (somente leitura)
- Inseridas no editor via botao de assinatura (disponivel no ResponseHUD)

### 4.4 Destaque de Solicitantes

Acesse via **Configuracoes > Destaque**.

- Busca qualquer pessoa no SMAX
- Adicione solicitantes a lista de destaque para identifica-los rapidamente nos chamados

### 4.5 Relatorio de Atividades

Acesse via **ResponseHUD > botao de relatorio** ou **Configuracoes > Respostas**.

- Gera relatorio por periodo (data inicio/fim)
- Resumo: chamados respondidos, vinculados, transferidos, designados, alteracoes de status
- Tabela detalhada com cada acao realizada
- Exportacao em CSV
- Sincronizacao automatica com o servidor

### 4.6 Consulta de Processos no eProc

Numeros de processo no formato CNJ sao detectados automaticamente em descricoes e discussoes.

**Formatos reconhecidos:**
- Formatado: `4000439-14.2026.8.26.0201`
- Bruto (20 digitos): `40004391420268260201`

Clique no numero e uma nova aba do eProc abre ja com a pesquisa executada (requer eProc aberto e logado).

---

## 5. Temas

Tres temas disponiveis, alternados pelo botao no canto superior do HUD:

| Tema | Descricao |
|------|-----------|
| Light | Fundo claro, texto escuro |
| Dark | Fundo escuro, texto claro |
| Gray | Tons neutros com acentos dourados |

A preferencia e salva e persiste entre sessoes.

---

## 6. Atalhos de teclado

| Tecla | Acao |
|-------|------|
| ESC | Fecha o painel ativo |
| Setas esquerda/direita | Navega entre imagens no visualizador de anexos |
| Enter | No campo de busca por numero, executa a pesquisa |

---

## 7. Atualizacoes

O script se atualiza automaticamente pelo Tampermonkey quando uma nova versao e publicada no repositorio. Para verificar manualmente:

1. Abra o Tampermonkey > **Painel de Controle**
2. Clique na aba **Utilitarios**
3. Clique em **Verificar atualizacoes**

---

## 8. Dicas

- Mantenha o cadastro de ausencias atualizado junto ao administrador -- chamados sao redistribuidos automaticamente
- Se um chamado cair na equipe errada, informe o administrador para revisar as regras de GSE e Local de Divulgacao
- Use o preset "Rejeitados" no ResponseHUD para localizar rapidamente chamados devolvidos
- O botao "Escalar" recoloca chamados rejeitados na fila de atendimento
- Utilize os filtros combinados para encontrar chamados rapidamente
- Scripts compartilhados sao atualizados automaticamente -- nao e necessario recria-los localmente
