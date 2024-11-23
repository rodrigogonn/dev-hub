---
slug: dynamodb-single-table-design
title: 'Entendendo o DynamoDB e o padrão Single Table Design'
authors: [rodrigo]
tags: [dynamodb, no-sql, aws, single-table-design]
date: 2024-11-23
description: 'Aprenda como funciona o DynamoDB e como estruturar dados com o padrão Single Table Design para eficiência e escalabilidade.'
---

## Por que o Single Table Design?

O DynamoDB é uma ferramenta poderosa, mas seu verdadeiro potencial está em como modelamos os dados. O padrão **Single Table Design** permite criar tabelas eficientes e altamente escaláveis. Neste post, você aprenderá a usar esse padrão na prática, desde os conceitos básicos até exemplos avançados.

<!--truncate-->

---

## O que é o DynamoDB?

O Amazon DynamoDB é um banco de dados NoSQL gerenciado pela AWS, projetado para **alta escalabilidade**, **baixa latência** e **alta disponibilidade**. Ele é ideal para aplicações que precisam lidar com grandes volumes de dados e acessos rápidos, como sistemas de e-commerce, jogos, aplicações móveis e APIs.

DynamoDB organiza os dados em tabelas, cada item sendo identificado por uma **chave primária**. O banco é altamente flexível, permitindo adicionar atributos diferentes para itens na mesma tabela.

---

## Principais Conceitos do DynamoDB

### Chaves Primárias

1. **Partition Key (PK):** Identifica de forma única uma partição onde os dados são armazenados.
2. **Sort Key (SK):** Opcional, organiza os dados dentro de uma mesma PK.

### Operações Básicas

- **Query:** Busca eficiente baseada na PK e opcionalmente filtrar pela SK. Retorna itens de uma mesma partição.
- **Scan:** Faz uma varredura em toda a tabela para encontrar itens que atendam a condições específicas. Geralmente menos eficiente que Query.

### Índices Secundários

1. **Global Secondary Index (GSI):** Permite criar uma nova visão da tabela, com uma PK e SK diferentes da tabela original.
2. **Local Secondary Index (LSI):** Usa a mesma PK, mas com uma SK alternativa. Ideal para consultas adicionais dentro de uma partição.

---

## Single Table Design: O Que é e Por Que Usar?

O **Single Table Design** (STD) é uma abordagem onde **todos os dados** de um sistema são organizados em uma única tabela, aproveitando a flexibilidade de PK e SK para criar relações hierárquicas e consultas eficientes.

### Vantagens:

- **Organização:** Facilita o agrupamento lógico de dados.
- **Consultas mais rápidas:** Reduz o número de operações e acessos ao banco.
- **Menor custo:** Menos tabelas significa menos custos e manutenção.

### Quando Usar?

- Sistemas com relações hierárquicas ou dados altamente relacionados.
- Necessidade de alta performance em consultas específicas.

---

## Design para um Blog com Posts e Comentários

### Estrutura da Tabela

| **PK**           | **SK**               | **Atributos**                      |
| ---------------- | -------------------- | ---------------------------------- |
| `author#rodrigo` | `metadata`           | `{ name: "Rodrigo Gonçalves" }`    |
| `author#rodrigo` | `post#123`           | `{ title: "Aprendendo DynamoDB" }` |
| `author#rodrigo` | `post#123#comment#1` | `{ text: "Ótimo post!" }`          |

---

## Consultas no DynamoDB

### 1. Listar posts de um autor

Queremos buscar todos os itens da tabela que começam com `post#` na SK para o autor `author#rodrigo`.

```javascript
const params = {
  TableName: 'BlogTable',
  KeyConditionExpression: 'PK = :authorId AND begins_with(SK, :postPrefix)',
  ExpressionAttributeValues: {
    ':authorId': 'author#rodrigo',
    ':postPrefix': 'post#',
  },
};

const result = await dynamoDB.query(params).promise();
console.log(result.Items);
```

---

### 2. Listar comentários de um post

Para buscar comentários relacionados a um post específico, usamos `PK = post#123` e verificamos a SK começando com `comment#`.

```javascript
const params = {
  TableName: 'BlogTable',
  KeyConditionExpression: 'PK = :postId AND begins_with(SK, :commentPrefix)',
  ExpressionAttributeValues: {
    ':postId': 'post#123',
    ':commentPrefix': 'comment#',
  },
};

const result = await dynamoDB.query(params).promise();
console.log(result.Items);
```

---

### 3. Buscar um post específico

Para encontrar um único post, usamos o identificador exato de `PK` e `SK`.

```javascript
const params = {
  TableName: 'BlogTable',
  KeyConditionExpression: 'PK = :authorId AND SK = :postId',
  ExpressionAttributeValues: {
    ':authorId': 'author#rodrigo',
    ':postId': 'post#123',
  },
};

const result = await dynamoDB.query(params).promise();
console.log(result.Items);
```

---

### 4. Buscar comentários ordenados por data usando um LSI

Se quisermos ordenar os comentários por data de criação, podemos criar um **Local Secondary Index (LSI)**, onde a SK alternativa é o atributo `createdAt`.

#### Estrutura com LSI:

| **PK**     | **SK**      | **createdAt**          |
| ---------- | ----------- | ---------------------- |
| `post#123` | `comment#1` | `2024-11-23T10:00:00Z` |
| `post#123` | `comment#2` | `2024-11-23T11:00:00Z` |

#### Consulta com LSI:

```javascript
const params = {
  TableName: 'BlogTable',
  IndexName: 'CreatedAtIndex', // Nome do LSI
  KeyConditionExpression: 'PK = :postId AND createdAt BETWEEN :start AND :end',
  ExpressionAttributeValues: {
    ':postId': 'post#123',
    ':start': '2024-11-23T00:00:00Z',
    ':end': '2024-11-23T23:59:59Z',
  },
};

const result = await dynamoDB.query(params).promise();
console.log(result.Items);
```

---

### 5. Buscar posts pelo título usando um GSI

Se quisermos buscar posts pelo título globalmente, podemos criar um **Global Secondary Index (GSI)**, onde a PK é o `title` e a SK pode ser o `authorId`.

#### Estrutura com GSI:

| **PK**           | **SK**     | **GSI PK: title**       |
| ---------------- | ---------- | ----------------------- |
| `author#rodrigo` | `post#123` | `"Aprendendo DynamoDB"` |

#### Consulta com GSI:

```javascript
const params = {
  TableName: 'BlogTable',
  IndexName: 'TitleIndex', // Nome do GSI
  KeyConditionExpression: 'title = :title',
  ExpressionAttributeValues: {
    ':title': 'Aprendendo DynamoDB',
  },
};

const result = await dynamoDB.query(params).promise();
console.log(result.Items);
```

---

## Conclusão

O DynamoDB, combinado com o padrão **Single Table Design**, oferece uma solução escalável e eficiente para aplicações modernas. Embora o design inicial possa parecer complexo, ele proporciona **flexibilidade**, **baixo custo** e **alta performance**.

Ao dominar operações como **Query**, **Scan** e a criação de **Índices Secundários**, você estará bem preparado para criar sistemas robustos e de alto desempenho.

Se você ficou com dúvidas ou quer discutir mais sobre este padrão, deixe um comentário ou entre em contato!

---
