# Modelagem DynamoDB - Sistema de Allowlists

## Visão Geral

Sistema de alta performance (5k RPS) para gerenciar allowlists de clientes com dois tipos:
- **FIXA**: Lista explícita de customers (pode ter milhões de IDs)
- **ONDA**: Regras dinâmicas sem lista de customers

## Arquitetura

```
API (40 pods × 150 RPS)
    ↓
Cache Local (LRU + Bloom Filters)
    ↓
DynamoDB + DAX
    ↓
DynamoDB Streams → Lambda → SNS/SQS (invalidação de cache)
```

## Estrutura da Tabela Principal

### Tabela: `Allowlists`

```
Partition Key: PK (String)
Sort Key: SK (String)
Billing Mode: On-Demand
Stream: Enabled (NEW_AND_OLD_IMAGES)
TTL: expiresAt (opcional)
```

### Padrões de Dados

```
┌─────────────────────────┬──────────────────────────┬────────────────────────────────────┐
│ PK                      │ SK                       │ Attributes                         │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#123           │ META                     │ name: "VIP Clients"                │
│                         │                          │ type: "FIXA"                       │
│                         │                          │ createdAt: "2025-01-01T00:00:00Z"  │
│                         │                          │ updatedAt: "2025-01-15T10:00:00Z"  │
│                         │                          │ customerCount: 5000000             │
│                         │                          │ version: 1                         │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#123           │ CUSTOMER#550e8400-...    │ addedAt: "2024-12-01T10:30:00Z"    │
│                         │                          │ addedBy: "admin@company.com"       │
│                         │                          │ metadata: {...}                    │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#123           │ CUSTOMER#660e8400-...    │ addedAt: "2024-12-02T11:00:00Z"    │
│                         │                          │ addedBy: "system"                  │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ...                     │ ...                      │ (milhões de customers)             │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#123           │ BLOOM_FILTER#v167890...  │ buffer: <Binary>                   │
│                         │                          │ itemCount: 5000000                 │
│                         │                          │ version: 1678901234                │
│                         │                          │ createdAt: "2025-01-15T10:00:00Z"  │
│                         │                          │ sizeBytes: 7200000                 │
│                         │                          │ compressed: true                   │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#456           │ META                     │ name: "Promo Wave Q1"              │
│                         │                          │ type: "ONDA"                       │
│                         │                          │ createdAt: "2025-02-01T00:00:00Z"  │
│                         │                          │ config: {                          │
│                         │                          │   strategy: "PERCENTAGE",          │
│                         │                          │   percentage: 0.3,                 │
│                         │                          │   startTime: 1704067200000,        │
│                         │                          │   endTime: 1711929600000           │
│                         │                          │ }                                  │
├─────────────────────────┼──────────────────────────┼────────────────────────────────────┤
│ ALLOWLIST#789           │ META                     │ name: "Beta Testers"               │
│                         │                          │ type: "ONDA"                       │
│                         │                          │ config: {                          │
│                         │                          │   strategy: "EXPLICIT_LIST",       │
│                         │                          │   customerIds: [...]               │
│                         │                          │ }                                  │
└─────────────────────────┴──────────────────────────┴────────────────────────────────────┘
```

### GSI: `CustomerAllowlistIndex` (Opcional)

**Uso:** Buscar todas allowlists que um customer pertence

```
Partition Key: GSI1PK (String)
Sort Key: GSI1SK (String)
Projection: KEYS_ONLY
```

```
┌─────────────────────────┬──────────────────────────┬────────────────────────────────────┐
│ GSI1PK                  │ GSI1SK                   │ PK              │ SK               │
├─────────────────────────┼──────────────────────────┼─────────────────┼──────────────────┤
│ CUSTOMER#550e8400-...   │ ALLOWLIST#123            │ ALLOWLIST#123   │ CUSTOMER#550e... │
├─────────────────────────┼──────────────────────────┼─────────────────┼──────────────────┤
│ CUSTOMER#550e8400-...   │ ALLOWLIST#999            │ ALLOWLIST#999   │ CUSTOMER#550e... │
└─────────────────────────┴──────────────────────────┴─────────────────┴──────────────────┘
```

## Conceito: PK Repetida no DynamoDB

### É possível e correto repetir a PK?

✅ **Sim!** No DynamoDB com composite key (PK + SK):
- A **chave primária real** é a combinação `PK + SK` (deve ser única)
- A `PK` sozinha **pode e deve se repetir**
- Todos os itens com a mesma `PK` ficam **fisicamente juntos** na mesma partição

```
PK              SK                    → Chave Primária Composta (única)
────────────────────────────────────────────────────────────────────
ALLOWLIST#123   META                  ✓ Único
ALLOWLIST#123   CUSTOMER#user001      ✓ Único
ALLOWLIST#123   CUSTOMER#user002      ✓ Único
ALLOWLIST#123   BLOOM_FILTER#v123     ✓ Único
```

### Query vs Scan - Performance

#### Query (O QUE USAMOS) ✅

```javascript
// Query: Busca APENAS na partição especificada
await dynamodb.query({
  TableName: 'Allowlists',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': 'ALLOWLIST#123',      // Vai direto na partição
    ':sk': 'CUSTOMER#'
  }
});
```

**Performance:**
- ⚡ **Rápido**: Acessa diretamente a partição `ALLOWLIST#123`
- 📊 **Eficiente**: Lê apenas os itens com `PK = ALLOWLIST#123`
- 💰 **Econômico**: Consome RCU proporcional aos dados lidos

#### Scan (NUNCA USAR PARA ISSO) ❌

```javascript
// Scan: Varre a TABELA INTEIRA
await dynamodb.scan({
  TableName: 'Allowlists',
  FilterExpression: 'PK = :pk',
  ExpressionAttributeValues: {
    ':pk': 'ALLOWLIST#123'
  }
});
```

**Performance:**
- 🐌 **Lento**: Lê TODA a tabela
- 📊 **Ineficiente**: Processa milhões de itens para filtrar alguns
- 💰 **Caro**: Consome RCU de TODA a tabela

### Limites do DynamoDB

#### 1. Limite de 10GB por Partição

Cada valor único de `PK` pode ter até **10GB de dados**:

```
ALLOWLIST#123 (uma partição):
- META: 1KB
- 5M customers × 200 bytes = 1GB
- Bloom Filter: 7MB
Total: ~1GB ✅ OK (bem abaixo de 10GB)
```

#### 2. Limite de 3000 RCU/WCU por Partição

Cada partição suporta até **3000 RCU/s** e **1000 WCU/s**.

**Nossa arquitetura:**
```
Total: 5000 RPS
Com cache (95% hit rate): ~250 RPS chegam no DynamoDB
Distribuído por ~100 allowlists diferentes: ~2-3 RPS por allowlist

✅ Muito abaixo do limite de 3000 RCU/partição
```

### Visualização da Estrutura Física

```
┌─────────────────────────────────────────────────────────────────┐
│ Partição: ALLOWLIST#123 (1GB, ~5M itens)                       │
├─────────────────────────────────────────────────────────────────┤
│ ALLOWLIST#123   META                                            │
│ ALLOWLIST#123   BLOOM_FILTER#v1678901234                        │
│ ALLOWLIST#123   CUSTOMER#550e8400-...                           │
│ ALLOWLIST#123   CUSTOMER#660e8400-...                           │
│ ...                                                              │
│ ALLOWLIST#123   CUSTOMER#999e8400-...                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Partição: ALLOWLIST#456 (2KB, 1 item)                          │
├─────────────────────────────────────────────────────────────────┤
│ ALLOWLIST#456   META                                            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Partição: ALLOWLIST#789 (1GB, ~5M itens)                       │
├─────────────────────────────────────────────────────────────────┤
│ ALLOWLIST#789   META                                            │
│ ALLOWLIST#789   BLOOM_FILTER#v1678905678                        │
│ ALLOWLIST#789   CUSTOMER#111e8400-...                           │
│ ...                                                              │
└─────────────────────────────────────────────────────────────────┘
```

**Query com `PK = ALLOWLIST#123`:**
- ✅ Vai **direto** na primeira partição
- ✅ Ignora todas as outras partições
- ✅ Eficiente e rápido

## Padrões de Acesso

### 1. Buscar Metadata da Allowlist

```javascript
const result = await dax.getItem({
  TableName: 'Allowlists',
  Key: {
    PK: 'ALLOWLIST#123',
    SK: 'META'
  }
});
```

**Performance:** ~1ms (via DAX), cache hit >99%

### 2. Verificar Customer na Allowlist (FIXA)

```javascript
// 1. Check Bloom Filter (in-memory)
if (!bloom.has(customerId)) {
  return false; // ~80% dos casos param aqui
}

// 2. Confirmação no DynamoDB
const result = await dax.getItem({
  TableName: 'Allowlists',
  Key: {
    PK: 'ALLOWLIST#123',
    SK: `CUSTOMER#${customerId}`
  }
});

return !!result.Item;
```

**Performance:**
- Bloom filter negativo: <1ms
- Bloom filter positivo + DynamoDB: ~3-5ms

### 3. Buscar Bloom Filter

```javascript
const result = await dax.query({
  TableName: 'Allowlists',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': 'ALLOWLIST#123',
    ':sk': 'BLOOM_FILTER#'
  },
  ScanIndexForward: false,  // Mais recente primeiro
  Limit: 1
});

const bloomData = result.Items[0];
const bloom = BloomFilter.fromJSON(
  bloomData.compressed
    ? zlib.gunzipSync(bloomData.buffer)
    : bloomData.buffer
);
```

### 4. Listar Customers (com paginação)

```javascript
const result = await dynamodb.query({
  TableName: 'Allowlists',
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': 'ALLOWLIST#123',
    ':sk': 'CUSTOMER#'
  },
  Limit: 1000,
  ExclusiveStartKey: lastEvaluatedKey
});

const customerIds = result.Items.map(item =>
  item.SK.replace('CUSTOMER#', '')
);
```

### 5. Buscar Allowlists de um Customer (GSI)

```javascript
const result = await dynamodb.query({
  TableName: 'Allowlists',
  IndexName: 'CustomerAllowlistIndex',
  KeyConditionExpression: 'GSI1PK = :pk',
  ExpressionAttributeValues: {
    ':pk': `CUSTOMER#${customerId}`
  }
});

const allowlistIds = result.Items.map(item =>
  item.GSI1SK.replace('ALLOWLIST#', '')
);
```

### 6. Adicionar Customer (Transação)

```javascript
await dynamodb.transactWriteItems({
  TransactItems: [
    {
      Put: {
        TableName: 'Allowlists',
        Item: {
          PK: 'ALLOWLIST#123',
          SK: `CUSTOMER#${customerId}`,
          GSI1PK: `CUSTOMER#${customerId}`,
          GSI1SK: 'ALLOWLIST#123',
          addedAt: new Date().toISOString(),
          addedBy: 'admin@company.com'
        }
      }
    },
    {
      Update: {
        TableName: 'Allowlists',
        Key: { PK: 'ALLOWLIST#123', SK: 'META' },
        UpdateExpression: 'SET customerCount = customerCount + :inc, updatedAt = :now',
        ExpressionAttributeValues: {
          ':inc': 1,
          ':now': new Date().toISOString()
        }
      }
    }
  ]
});
```

### 7. Remover Customer (Transação)

```javascript
await dynamodb.transactWriteItems({
  TransactItems: [
    {
      Delete: {
        TableName: 'Allowlists',
        Key: {
          PK: 'ALLOWLIST#123',
          SK: `CUSTOMER#${customerId}`
        }
      }
    },
    {
      Update: {
        TableName: 'Allowlists',
        Key: { PK: 'ALLOWLIST#123', SK: 'META' },
        UpdateExpression: 'SET customerCount = customerCount - :dec, updatedAt = :now',
        ExpressionAttributeValues: {
          ':dec': 1,
          ':now': new Date().toISOString()
        }
      }
    }
  ]
});
```

### 8. Batch Get Metadata de Múltiplas Allowlists

```javascript
const result = await dax.batchGetItem({
  RequestItems: {
    'Allowlists': {
      Keys: [
        { PK: 'ALLOWLIST#123', SK: 'META' },
        { PK: 'ALLOWLIST#456', SK: 'META' },
        { PK: 'ALLOWLIST#789', SK: 'META' }
      ]
    }
  }
});

const allowlists = result.Responses.Allowlists;
```

## Implementação da API

### Endpoint: `GET /customers/:customerId/check?allowlists=123,456`

```javascript
class AllowlistChecker {
  async checkMultiple(customerId, allowlistIds) {
    // Consulta paralela
    const results = await Promise.all(
      allowlistIds.map(id => this.checkSingle(customerId, id))
    );

    return {
      customerId,
      results: allowlistIds.map((id, index) => ({
        allowlistId: id,
        allowed: results[index].allowed,
        type: results[index].type,
        checkedAt: new Date().toISOString()
      })),
      overallAllowed: results.every(r => r.allowed)
    };
  }

  async checkSingle(customerId, allowlistId) {
    // 1. Cache de verificação (TTL 1min)
    const cacheKey = `${allowlistId}:${customerId}`;
    const cached = verificationCache.get(cacheKey);
    if (cached !== undefined) {
      metrics.recordCacheHit();
      return cached;
    }

    // 2. Buscar metadata (quase sempre em cache DAX)
    const meta = await this.getMetadata(allowlistId);

    // 3. Despachar por tipo
    let result;
    if (meta.type === 'FIXA') {
      result = await this.checkFixa(allowlistId, customerId, meta);
    } else {
      result = await this.checkOnda(allowlistId, customerId, meta);
    }

    verificationCache.set(cacheKey, result, 60);
    return result;
  }

  async checkFixa(allowlistId, customerId, meta) {
    // 1. Carregar Bloom Filter (lazy)
    let bloom = bloomFilters.get(allowlistId);
    if (!bloom) {
      bloom = await this.loadBloomFilter(allowlistId);
      bloomFilters.set(allowlistId, bloom);
    }

    // 2. Bloom Filter check
    if (!bloom.has(customerId)) {
      metrics.recordBloomFiltered();
      return { allowed: false, type: 'FIXA' };
    }

    // 3. Verificação no DynamoDB via DAX
    const response = await dax.getItem({
      TableName: 'Allowlists',
      Key: {
        PK: `ALLOWLIST#${allowlistId}`,
        SK: `CUSTOMER#${customerId}`
      }
    });

    return {
      allowed: !!response.Item,
      type: 'FIXA',
      addedAt: response.Item?.addedAt
    };
  }

  async checkOnda(allowlistId, customerId, meta) {
    // Avaliação local de regras
    const allowed = this.evaluateOndaRules(meta.config, customerId);

    return {
      allowed,
      type: 'ONDA',
      rules: meta.config
    };
  }

  evaluateOndaRules(config, customerId) {
    if (config.strategy === 'PERCENTAGE') {
      const hash = this.hashCustomerId(customerId);
      return hash < config.percentage;
    }

    if (config.strategy === 'TIME_WINDOW') {
      const now = Date.now();
      return now >= config.startTime && now <= config.endTime;
    }

    if (config.strategy === 'EXPLICIT_LIST') {
      return config.customerIds.includes(customerId);
    }

    return false;
  }
}
```

### Exemplo de Request/Response

**Request:**
```http
GET /customers/550e8400-e29b-4d4e-a164-426655440000/check?allowlists=123,456
```

**Fluxo de Execução (paralelo):**

```javascript
// Allowlist 123 (FIXA):
// 1. Get metadata (DAX, ~1ms)
// 2. Bloom filter check (in-memory, <1ms)
// 3. DynamoDB confirmation (DAX, ~2ms)

// Allowlist 456 (ONDA):
// 1. Get metadata (DAX, ~1ms)
// 2. Evaluate rules (in-memory, <1ms)

// Total: ~3-5ms
```

**Response:**
```json
{
  "customerId": "550e8400-e29b-4d4e-a164-426655440000",
  "results": [
    {
      "allowlistId": "123",
      "allowed": true,
      "type": "FIXA",
      "checkedAt": "2025-10-06T14:32:15.123Z"
    },
    {
      "allowlistId": "456",
      "allowed": true,
      "type": "ONDA",
      "checkedAt": "2025-10-06T14:32:15.125Z"
    }
  ],
  "overallAllowed": true
}
```

## Bloom Filter - Geração e Sincronização

### Estrutura no DynamoDB

```javascript
{
  PK: 'ALLOWLIST#123',
  SK: 'BLOOM_FILTER#v1678901234',
  buffer: <Binary>,           // Bloom filter comprimido
  itemCount: 5000000,         // Número de customers
  version: 1678901234,        // Timestamp para versionamento
  createdAt: '2025-01-15T10:00:00Z',
  sizeBytes: 7200000,         // ~7MB comprimido
  compressed: true,
  falsePositiveRate: 0.0001   // 0.01%
}
```

### Geração (Processo Batch)

```javascript
class BloomFilterManager {
  async rebuildBloomFilter(allowlistId) {
    console.log(`Rebuilding bloom filter for ${allowlistId}`);

    // 1. Scan todos customers
    const customers = [];
    let lastKey = null;

    do {
      const result = await dynamodb.query({
        TableName: 'Allowlists',
        KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
        ExpressionAttributeValues: {
          ':pk': `ALLOWLIST#${allowlistId}`,
          ':sk': 'CUSTOMER#'
        },
        ExclusiveStartKey: lastKey,
        Limit: 10000
      });

      customers.push(...result.Items.map(i => i.SK.split('#')[1]));
      lastKey = result.LastEvaluatedKey;

      console.log(`Loaded ${customers.length} customers...`);
    } while (lastKey);

    // 2. Criar Bloom Filter
    const bloom = BloomFilter.create(customers.length, 0.0001);
    customers.forEach(id => bloom.add(id));

    // 3. Comprimir
    const json = bloom.saveAsJSON();
    const compressed = zlib.gzipSync(Buffer.from(json));

    // 4. Salvar no DynamoDB
    const version = Date.now();
    await dynamodb.putItem({
      TableName: 'Allowlists',
      Item: {
        PK: `ALLOWLIST#${allowlistId}`,
        SK: `BLOOM_FILTER#v${version}`,
        buffer: compressed,
        itemCount: customers.length,
        version: version,
        createdAt: new Date().toISOString(),
        sizeBytes: compressed.length,
        compressed: true,
        falsePositiveRate: 0.0001
      }
    });

    // 5. Notificar pods via SNS
    await sns.publish({
      TopicArn: process.env.BLOOM_UPDATE_TOPIC,
      Message: JSON.stringify({
        type: 'BLOOM_UPDATED',
        allowlistId: allowlistId,
        version: version,
        itemCount: customers.length
      })
    });

    console.log(`Bloom filter rebuilt: ${customers.length} items, ${compressed.length} bytes`);
  }
}
```

### Carregamento no Pod

```javascript
async loadBloomFilter(allowlistId) {
  // Buscar versão mais recente
  const result = await dax.query({
    TableName: 'Allowlists',
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues: {
      ':pk': `ALLOWLIST#${allowlistId}`,
      ':sk': 'BLOOM_FILTER#'
    },
    ScanIndexForward: false,
    Limit: 1
  });

  if (!result.Items?.length) {
    throw new Error(`Bloom filter not found for ${allowlistId}`);
  }

  const data = result.Items[0];

  // Descomprimir
  const buffer = data.compressed
    ? zlib.gunzipSync(data.buffer)
    : data.buffer;

  const bloom = BloomFilter.fromJSON(buffer.toString());

  console.log(`Loaded bloom filter: ${data.itemCount} items, ${data.sizeBytes} bytes`);

  return bloom;
}
```

### Sincronização via DynamoDB Streams

**Lambda que processa stream:**

```javascript
exports.handler = async (event) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT' || record.eventName === 'MODIFY') {
      const item = AWS.DynamoDB.Converter.unmarshall(record.dynamodb.NewImage);

      // Customer adicionado/removido → invalidar cache
      if (item.SK.startsWith('CUSTOMER#')) {
        await sns.publish({
          TopicArn: process.env.INVALIDATION_TOPIC,
          Message: JSON.stringify({
            type: 'CUSTOMER_CHANGED',
            allowlistId: item.PK.split('#')[1],
            customerId: item.SK.split('#')[1],
            action: record.eventName
          })
        });
      }

      // Bloom filter atualizado → notificar pods
      if (item.SK.startsWith('BLOOM_FILTER#')) {
        await sns.publish({
          TopicArn: process.env.INVALIDATION_TOPIC,
          Message: JSON.stringify({
            type: 'BLOOM_UPDATED',
            allowlistId: item.PK.split('#')[1],
            version: item.version
          })
        });
      }
    }
  }
};
```

**Pod recebe notificação (SQS):**

```javascript
class CacheSyncService {
  constructor(cache) {
    this.cache = cache;
    this.sqsUrl = process.env.CACHE_INVALIDATION_QUEUE;
    this.pollQueue();
  }

  async pollQueue() {
    while (true) {
      try {
        const messages = await sqs.receiveMessage({
          QueueUrl: this.sqsUrl,
          WaitTimeSeconds: 20,
          MaxNumberOfMessages: 10
        });

        for (const msg of messages.Messages || []) {
          const event = JSON.parse(JSON.parse(msg.Body).Message);

          switch (event.type) {
            case 'BLOOM_UPDATED':
              // Remove do cache para forçar reload
              this.cache.bloomFilters.delete(event.allowlistId);
              console.log(`Invalidated bloom filter: ${event.allowlistId}`);
              break;

            case 'CUSTOMER_CHANGED':
              // Invalida verificações específicas
              const key = `${event.allowlistId}:${event.customerId}`;
              this.cache.verificationCache.del(key);
              break;

            case 'ALLOWLIST_DELETED':
              // Limpa tudo relacionado
              this.invalidateAllowlist(event.allowlistId);
              break;
          }

          await sqs.deleteMessage({
            QueueUrl: this.sqsUrl,
            ReceiptHandle: msg.ReceiptHandle
          });
        }
      } catch (error) {
        console.error('Error polling SQS:', error);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }

  invalidateAllowlist(allowlistId) {
    this.cache.bloomFilters.delete(allowlistId);
    this.cache.metaCache.del(allowlistId);

    // Remove todas verificações dessa allowlist
    const keys = this.cache.verificationCache.keys();
    keys.forEach(key => {
      if (key.startsWith(`${allowlistId}:`)) {
        this.cache.verificationCache.del(key);
      }
    });
  }
}
```

## Estratégia de Cache por Pod

### Estrutura do Cache Local

```javascript
const NodeCache = require('node-cache');
const { BloomFilter } = require('bloom-filters');

class AllowlistCache {
  constructor() {
    // Cache de metadata (pequeno, TTL 5min)
    this.metaCache = new NodeCache({
      stdTTL: 300,
      maxKeys: 10000,
      checkperiod: 60
    });

    // Bloom Filters (in-memory, ~1MB por milhão de IDs)
    this.bloomFilters = new Map();

    // Cache de verificações recentes (LRU, TTL 1min)
    this.verificationCache = new NodeCache({
      stdTTL: 60,
      maxKeys: 50000,  // ~150 RPS * 60s * fator de boost
      checkperiod: 10
    });
  }

  getStats() {
    return {
      metaCacheSize: this.metaCache.keys().length,
      bloomFiltersCount: this.bloomFilters.size,
      verificationCacheSize: this.verificationCache.keys().length,
      estimatedMemoryMB: this.estimateMemoryUsage()
    };
  }

  estimateMemoryUsage() {
    let total = 0;

    // Metadata: ~1KB por allowlist
    total += this.metaCache.keys().length * 1024;

    // Bloom filters: ~1.5MB por milhão de IDs
    this.bloomFilters.forEach((bloom, id) => {
      total += 1.5 * 1024 * 1024; // Estimativa
    });

    // Verification cache: ~200 bytes por entry
    total += this.verificationCache.keys().length * 200;

    return (total / 1024 / 1024).toFixed(2);
  }
}
```

### Warmup do Pod na Inicialização

```javascript
class PodInitializer {
  constructor(cache) {
    this.cache = cache;
  }

  async warmup() {
    console.log('Starting cache warmup...');

    // 1. Buscar top allowlists (por frequência de acesso)
    const topAllowlists = await this.getTopAllowlists();

    // 2. Pré-carregar em paralelo
    await Promise.all(topAllowlists.map(async (allowlist) => {
      try {
        // Carregar metadata
        const meta = await this.fetchMetadata(allowlist.id);
        this.cache.metaCache.set(allowlist.id, meta);

        // Se FIXA, pré-carregar Bloom Filter
        if (meta.type === 'FIXA') {
          const bloom = await this.loadBloomFilter(allowlist.id);
          this.cache.bloomFilters.set(allowlist.id, bloom);
          console.log(`Warmed bloom filter: ${allowlist.id} (${meta.customerCount} items)`);
        }
      } catch (error) {
        console.error(`Failed to warm ${allowlist.id}:`, error);
      }
    }));

    console.log(`Warmup complete: ${topAllowlists.length} allowlists loaded`);
    console.log('Cache stats:', this.cache.getStats());
  }

  async getTopAllowlists() {
    // Buscar do Redis/Parameter Store/S3
    // Exemplo: top 10 allowlists mais acessadas
    return [
      { id: '123', frequency: 0.45 },
      { id: '456', frequency: 0.30 },
      { id: '789', frequency: 0.15 }
    ];
  }

  async fetchMetadata(allowlistId) {
    const result = await dax.getItem({
      TableName: 'Allowlists',
      Key: {
        PK: `ALLOWLIST#${allowlistId}`,
        SK: 'META'
      }
    });
    return result.Item;
  }

  async loadBloomFilter(allowlistId) {
    const result = await dax.query({
      TableName: 'Allowlists',
      KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
      ExpressionAttributeValues: {
        ':pk': `ALLOWLIST#${allowlistId}`,
        ':sk': 'BLOOM_FILTER#'
      },
      ScanIndexForward: false,
      Limit: 1
    });

    const data = result.Items[0];
    const buffer = data.compressed
      ? zlib.gunzipSync(data.buffer)
      : data.buffer;

    return BloomFilter.fromJSON(buffer.toString());
  }
}

// Uso no startup do pod
const cache = new AllowlistCache();
const initializer = new PodInitializer(cache);

app.listen(3000, async () => {
  console.log('Server starting on port 3000...');
  await initializer.warmup();
  console.log('Server ready to accept requests');
});
```

## Métricas e Monitoramento

```javascript
class MetricsCollector {
  constructor() {
    this.metrics = {
      cacheHits: 0,
      cacheMisses: 0,
      bloomFiltered: 0,
      dynamoQueries: 0,
      latencies: []
    };

    // Flush para CloudWatch a cada minuto
    this.flushInterval = setInterval(() => this.flush(), 60000);
  }

  recordCacheHit() {
    this.metrics.cacheHits++;
  }

  recordCacheMiss() {
    this.metrics.cacheMisses++;
  }

  recordBloomFiltered() {
    this.metrics.bloomFiltered++;
  }

  recordDynamoQuery() {
    this.metrics.dynamoQueries++;
  }

  recordLatency(ms) {
    this.metrics.latencies.push(ms);
  }

  async flush() {
    const total = this.metrics.cacheHits + this.metrics.cacheMisses;
    const cacheHitRate = total > 0 ? (this.metrics.cacheHits / total) * 100 : 0;
    const bloomEfficiency = this.metrics.cacheMisses > 0
      ? (this.metrics.bloomFiltered / this.metrics.cacheMisses) * 100
      : 0;

    const avgLatency = this.metrics.latencies.length > 0
      ? this.metrics.latencies.reduce((a, b) => a + b, 0) / this.metrics.latencies.length
      : 0;

    const p95Latency = this.calculatePercentile(this.metrics.latencies, 0.95);

    await cloudwatch.putMetricData({
      Namespace: 'Allowlist/Pod',
      MetricData: [
        {
          MetricName: 'CacheHitRate',
          Value: cacheHitRate,
          Unit: 'Percent',
          Dimensions: [{ Name: 'PodId', Value: process.env.HOSTNAME }]
        },
        {
          MetricName: 'BloomFilterEfficiency',
          Value: bloomEfficiency,
          Unit: 'Percent',
          Dimensions: [{ Name: 'PodId', Value: process.env.HOSTNAME }]
        },
        {
          MetricName: 'DynamoQueriesPerMinute',
          Value: this.metrics.dynamoQueries,
          Unit: 'Count',
          Dimensions: [{ Name: 'PodId', Value: process.env.HOSTNAME }]
        },
        {
          MetricName: 'AverageLatency',
          Value: avgLatency,
          Unit: 'Milliseconds',
          Dimensions: [{ Name: 'PodId', Value: process.env.HOSTNAME }]
        },
        {
          MetricName: 'P95Latency',
          Value: p95Latency,
          Unit: 'Milliseconds',
          Dimensions: [{ Name: 'PodId', Value: process.env.HOSTNAME }]
        }
      ]
    });

    // Log local
    console.log('Metrics:', {
      cacheHitRate: `${cacheHitRate.toFixed(2)}%`,
      bloomEfficiency: `${bloomEfficiency.toFixed(2)}%`,
      dynamoQueries: this.metrics.dynamoQueries,
      avgLatency: `${avgLatency.toFixed(2)}ms`,
      p95Latency: `${p95Latency.toFixed(2)}ms`
    });

    // Reset
    this.metrics = {
      cacheHits: 0,
      cacheMisses: 0,
      bloomFiltered: 0,
      dynamoQueries: 0,
      latencies: []
    };
  }

  calculatePercentile(arr, p) {
    if (arr.length === 0) return 0;
    const sorted = arr.sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * p) - 1;
    return sorted[index];
  }

  destroy() {
    clearInterval(this.flushInterval);
  }
}
```

## Customer em Múltiplas Allowlists

### Cenário: API recebe allowlists como parâmetro

```http
GET /customers/550e8400.../check?allowlists=123,456,789
```

**Verificação paralela (não precisa de GSI):**

```javascript
async checkMultiple(customerId, allowlistIds) {
  // Verifica todas em paralelo
  const results = await Promise.all(
    allowlistIds.map(id => this.checkSingle(customerId, id))
  );

  // 3 allowlists = 3 queries paralelas
  // Latência total: ~3-5ms (não 3×5ms, porque é paralelo)
}
```

**Performance:**
- 3 allowlists FIXA: 3× Bloom check + ~0-3 queries DynamoDB
- 3 allowlists ONDA: 0 queries DynamoDB (avaliação local)
- Latência: ~3-5ms total (paralelo)

### Quando usar o GSI CustomerAllowlistIndex?

Use o GSI se precisar responder: **"Quais allowlists esse customer pertence?"**

```javascript
// Query no GSI
const result = await dynamodb.query({
  TableName: 'Allowlists',
  IndexName: 'CustomerAllowlistIndex',
  KeyConditionExpression: 'GSI1PK = :pk',
  ExpressionAttributeValues: {
    ':pk': 'CUSTOMER#550e8400-...'
  }
});

// Se customer está em 50 allowlists:
// - Retorna 50 itens
// - Consome: ~1 RCU (50 × 100 bytes KEYS_ONLY)
// - Latência: ~5-10ms
```

## Allowlists com Milhões de UUIDs

Para allowlists muito grandes (2M-100M+ customers), temos estratégias de otimização:

### Estratégia A: Particionamento por Hash (Sharding)

**Quando usar:** Allowlist com >10M customers ou queries muito frequentes (hot partition)

```javascript
// Divide em N partições (shards)
function getShardKey(allowlistId, customerId, numShards = 10) {
  const hash = crypto.createHash('md5')
    .update(customerId)
    .digest('hex');
  const shardId = parseInt(hash.substring(0, 8), 16) % numShards;

  return {
    PK: `ALLOWLIST#${allowlistId}#SHARD_${shardId}`,
    SK: `CUSTOMER#${customerId}`
  };
}

// Estrutura no DynamoDB:
ALLOWLIST#123#SHARD_0   CUSTOMER#550e8400-...
ALLOWLIST#123#SHARD_0   CUSTOMER#660e8400-...
ALLOWLIST#123#SHARD_1   CUSTOMER#770e8400-...
...
ALLOWLIST#123#SHARD_9   CUSTOMER#990e8400-...

// Metadata da allowlist armazena número de shards
ALLOWLIST#123   META   {
  name: "VIP Giant",
  type: "FIXA",
  customerCount: 50000000,
  shardCount: 50  // 50 shards × 1M customers = 50M
}

// Query (você sabe exatamente qual shard buscar)
async checkFixaWithSharding(allowlistId, customerId) {
  const numShards = await this.getShardCount(allowlistId); // Ex: 10
  const { PK, SK } = getShardKey(allowlistId, customerId, numShards);

  // 1. Bloom filter ainda funciona (um por shard ou global)
  const bloom = await this.loadBloomFilter(allowlistId);
  if (!bloom.has(customerId)) {
    return false;
  }

  // 2. Query no shard específico
  const result = await dax.getItem({
    TableName: 'Allowlists',
    Key: { PK, SK }
  });

  return !!result.Item;
}
```

**Vantagens:**
- ✅ Cada partição: <10GB (limite DynamoDB)
- ✅ Distribui carga de leitura (evita hot partition)
- ✅ Query continua rápida (vai direto no shard)
- ✅ Bloom filter continua funcionando

**Desvantagens:**
- ❌ Listar todos customers fica mais complexo (precisa query em N shards)
- ❌ Adicionar customer precisa calcular shard

### Estratégia B: Bloom Filter em Camadas (Hierárquico)

**Quando usar:** Allowlist gigante (50M+) com alta taxa de verificação

```javascript
// Estrutura: Bloom pequeno (verificação rápida) + Bloom completo
ALLOWLIST#123   META                    { type: "FIXA", ... }
ALLOWLIST#123   BLOOM_FILTER#L1         { buffer: <100KB>, coverage: 0.01 }
ALLOWLIST#123   BLOOM_FILTER#L2         { buffer: <10MB>, coverage: 1.0 }

async checkFixaLayered(allowlistId, customerId) {
  // 1. L1 Bloom (carregado em todos os pods, ~100KB)
  const bloomL1 = await this.loadBloomFilterL1(allowlistId);
  if (!bloomL1.has(customerId)) {
    return false; // 99% dos "não está" param aqui
  }

  // 2. L2 Bloom (lazy load, ~10MB)
  const bloomL2 = await this.loadBloomFilterL2(allowlistId);
  if (!bloomL2.has(customerId)) {
    return false; // Mais 0.9% param aqui
  }

  // 3. DynamoDB (apenas 0.1% chegam aqui)
  const result = await dax.getItem({
    Key: {
      PK: `ALLOWLIST#${allowlistId}`,
      SK: `CUSTOMER#${customerId}`
    }
  });

  return !!result.Item;
}
```

**Vantagens:**
- ✅ L1 minúsculo (todos pods carregam na memória)
- ✅ L2 sob demanda (só carrega se L1 retornar "maybe")
- ✅ Reduz tráfego DynamoDB drasticamente

### Estratégia C: Hybrid - ElastiCache Redis Sets

**Quando usar:** Allowlist super crítica com altíssimo tráfego (>1k RPS só nela)

```javascript
// Redis: SET por allowlist
SADD allowlist:123 "550e8400-..." "660e8400-..." ...

async checkFixaRedis(allowlistId, customerId) {
  // 1. Check no Redis (O(1), <1ms)
  const isMember = await redis.sismember(
    `allowlist:${allowlistId}`,
    customerId
  );

  if (isMember === 1) return true;
  if (isMember === 0) return false;

  // 2. Fallback para DynamoDB se Redis falhar
  return this.checkFixaDynamoDB(allowlistId, customerId);
}

// Sincronização: DynamoDB Streams → Lambda → Redis
exports.streamHandler = async (event) => {
  for (const record of event.Records) {
    const item = unmarshall(record.dynamodb.NewImage);

    if (item.SK.startsWith('CUSTOMER#')) {
      const allowlistId = item.PK.split('#')[1];
      const customerId = item.SK.split('#')[1];

      if (record.eventName === 'INSERT') {
        await redis.sadd(`allowlist:${allowlistId}`, customerId);
      } else if (record.eventName === 'REMOVE') {
        await redis.srem(`allowlist:${allowlistId}`, customerId);
      }
    }
  }
};
```

**Vantagens:**
- ✅ Latência consistente <1ms (tudo em memória)
- ✅ Suporta milhões de customers sem problema
- ✅ `SISMEMBER` é O(1)

**Desvantagens:**
- ❌ Custo adicional do Redis
- ❌ Complexidade de sincronização
- ❌ Memória: 50M UUIDs × 36 bytes = ~1.8GB por allowlist

### Estratégia D: Tabela Separada para Allowlists Gigantes

**Quando usar:** Poucas allowlists gigantes (>50M), muitas pequenas

```javascript
// Tabela 1: Allowlists normais (<10M)
Table: Allowlists
PK: ALLOWLIST#123   SK: CUSTOMER#...

// Tabela 2: Allowlists gigantes (particionadas)
Table: AllowlistsGiant
PK: ALLOWLIST#999#SHARD_0   SK: CUSTOMER#...

// Metadata indica qual tabela usar
{
  PK: 'ALLOWLIST#999',
  SK: 'META',
  type: 'FIXA',
  customerCount: 100000000,
  storage: 'GIANT_TABLE',  // <-- Flag
  shardCount: 100
}

// Roteamento automático
async checkSingle(customerId, allowlistId) {
  const meta = await this.getMetadata(allowlistId);

  if (meta.storage === 'GIANT_TABLE') {
    return this.checkGiantTable(customerId, allowlistId, meta);
  }

  return this.checkNormalTable(customerId, allowlistId, meta);
}
```

**Vantagens:**
- ✅ Allowlists normais não são afetadas
- ✅ Throughput isolado (não compete RCU)
- ✅ Pode ter billing mode diferente (ex: Provisioned com reserved capacity)

### Recomendação por Tamanho

| Customers        | Estratégia Recomendada                | Bloom Filter | Custo/Complexidade |
|------------------|---------------------------------------|--------------|---------------------|
| < 5M             | **Partição única** (atual)            | 1× 7MB       | 💚 Baixo            |
| 5M - 20M         | **Partição única** + Bloom L2         | 15MB total   | 💚 Baixo            |
| 20M - 50M        | **Sharding (10 shards)**              | 10× 7MB      | 💛 Médio            |
| 50M - 100M       | **Sharding (50 shards)** + Redis L1   | Híbrido      | 🧡 Alto             |
| > 100M           | **Tabela separada** + Redis completo  | Redis only   | ❤️ Muito Alto       |

### Implementação Prática: Sharding

```javascript
class AllowlistChecker {
  async checkSingle(customerId, allowlistId) {
    // 1. Cache verificação
    const cacheKey = `${allowlistId}:${customerId}`;
    const cached = verificationCache.get(cacheKey);
    if (cached !== undefined) return cached;

    // 2. Metadata
    const meta = await this.getMetadata(allowlistId);

    // 3. Despachar por tipo e tamanho
    let result;
    if (meta.type === 'ONDA') {
      result = await this.checkOnda(allowlistId, customerId, meta);
    } else if (meta.shardCount > 1) {
      result = await this.checkFixaSharded(allowlistId, customerId, meta);
    } else {
      result = await this.checkFixa(allowlistId, customerId, meta);
    }

    verificationCache.set(cacheKey, result, 60);
    return result;
  }

  async checkFixaSharded(allowlistId, customerId, meta) {
    const shardCount = meta.shardCount;

    // 1. Bloom Filter global (ou por shard)
    let bloom = bloomFilters.get(allowlistId);
    if (!bloom) {
      bloom = await this.loadBloomFilter(allowlistId);
      bloomFilters.set(allowlistId, bloom);
    }

    if (!bloom.has(customerId)) {
      return { allowed: false, type: 'FIXA' };
    }

    // 2. Calcular shard
    const hash = crypto.createHash('md5')
      .update(customerId)
      .digest('hex');
    const shardId = parseInt(hash.substring(0, 8), 16) % shardCount;

    // 3. Query no shard específico
    const result = await dax.getItem({
      TableName: 'Allowlists',
      Key: {
        PK: `ALLOWLIST#${allowlistId}#SHARD_${shardId}`,
        SK: `CUSTOMER#${customerId}`
      }
    });

    return {
      allowed: !!result.Item,
      type: 'FIXA',
      shard: shardId
    };
  }
}
```

### Geração de Bloom Filter para Allowlist Shardada

```javascript
async function rebuildBloomFilterSharded(allowlistId, shardCount) {
  console.log(`Building bloom filter for ${allowlistId} (${shardCount} shards)`);

  const allCustomers = [];

  // 1. Scan todos os shards em paralelo
  const shardPromises = Array.from({ length: shardCount }, async (_, i) => {
    const customers = [];
    let lastKey = null;

    do {
      const result = await dynamodb.query({
        TableName: 'Allowlists',
        KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
        ExpressionAttributeValues: {
          ':pk': `ALLOWLIST#${allowlistId}#SHARD_${i}`,
          ':sk': 'CUSTOMER#'
        },
        ExclusiveStartKey: lastKey,
        Limit: 10000
      });

      customers.push(...result.Items.map(item => item.SK.split('#')[1]));
      lastKey = result.LastEvaluatedKey;
    } while (lastKey);

    console.log(`Shard ${i}: ${customers.length} customers`);
    return customers;
  });

  const shardsData = await Promise.all(shardPromises);
  shardsData.forEach(customers => allCustomers.push(...customers));

  console.log(`Total: ${allCustomers.length} customers`);

  // 2. Criar Bloom Filter global
  const bloom = BloomFilter.create(allCustomers.length, 0.0001);
  allCustomers.forEach(id => bloom.add(id));

  // 3. Comprimir e salvar
  const compressed = zlib.gzipSync(Buffer.from(bloom.saveAsJSON()));

  await dynamodb.putItem({
    TableName: 'Allowlists',
    Item: {
      PK: `ALLOWLIST#${allowlistId}`,  // Sem shard (é global)
      SK: `BLOOM_FILTER#v${Date.now()}`,
      buffer: compressed,
      itemCount: allCustomers.length,
      shardCount: shardCount
    }
  });

  console.log(`Bloom filter saved: ${compressed.length} bytes`);
}
```

### Decisão Simplificada

Para a maioria dos casos (até 20M customers por allowlist):

```javascript
// Mantenha o design atual (partição única)
// Adicione apenas um check de tamanho

async checkFixa(allowlistId, customerId, meta) {
  // Se muito grande, alerta (mas continua funcionando)
  if (meta.customerCount > 20000000) {
    logger.warn(`Large allowlist: ${allowlistId} (${meta.customerCount} customers)`);
    // Considere migrar para sharding no futuro
  }

  // Fluxo normal: Bloom → DynamoDB
  let bloom = bloomFilters.get(allowlistId);
  if (!bloom) {
    bloom = await this.loadBloomFilter(allowlistId);
    bloomFilters.set(allowlistId, bloom);
  }

  if (!bloom.has(customerId)) {
    return { allowed: false, type: 'FIXA' };
  }

  const result = await dax.getItem({
    TableName: 'Allowlists',
    Key: {
      PK: `ALLOWLIST#${allowlistId}`,
      SK: `CUSTOMER#${customerId}`
    }
  });

  return { allowed: !!result.Item, type: 'FIXA' };
}
```

**Quando migrar para sharding:**
- Allowlist ultrapassa 10GB (limite físico)
- Queries throttled (hot partition >3000 RPS)
- Bloom filter > 50MB (impacta memória dos pods)

## Estimativas de Performance e Custos

### Performance por Pod (150 RPS)

```
Distribuição de Requests:
- Cache local hit: ~80% (120 RPS) → latência <1ms
- Bloom filter (não está): ~15% (22 RPS) → latência <2ms
- DynamoDB+DAX (confirma): ~5% (8 RPS) → latência ~3-5ms

Total cluster (40 pods × 150 RPS = 6000 RPS):
- 4800 req/s servidas por cache local (0 custo DynamoDB)
- 1200 req/s vão para Bloom/DynamoDB
  - 900 req/s filtradas por Bloom (0 custo DynamoDB)
  - 300 req/s chegam no DynamoDB (facilmente suportado por DAX)
```

### Memória por Pod

```
Componentes:
- Metadata cache: ~10MB (10k allowlists × 1KB)
- Bloom filters: ~50MB (10 allowlists × 5M customers)
- Verification cache: ~10MB (50k entries × 200 bytes)

Total: ~70MB overhead por pod
Total cluster: 40 pods × 70MB = 2.8GB
```

### Custos Mensais (estimativa)

```
DynamoDB:
- Storage: 10GB × $0.25 = $2.50
- On-demand reads: 300 RPS × 2.6M seg/mês × $0.25/1M = $195
- On-demand writes: 10 WPS × 2.6M seg/mês × $1.25/1M = $32.50

DAX (3 nodes r5.large):
- Instâncias: 3 × $0.228/hr × 730hr = $499

ElastiCache Redis (opcional, cache.t3.medium):
- Instância: $0.068/hr × 730hr = $50

Total estimado: ~$780/mês

Com cache otimizado (hit rate >95%):
- DynamoDB reads: ~$10-20
- Total: ~$580/mês
```

### Armazenamento

```
Allowlist FIXA (5M customers):
- META: ~1KB
- Customers: 5M × 200 bytes = ~1GB
- Bloom Filter: ~7MB comprimido
Total: ~1GB

Allowlist ONDA:
- META com config: ~2KB

10 Allowlists FIXA + 50 ONDA:
- Total: ~10GB
- Custo storage: ~$2.50/mês
```

## Infraestrutura como Código

### Terraform

```hcl
# DynamoDB Table
resource "aws_dynamodb_table" "allowlists" {
  name           = "Allowlists"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "PK"
  range_key      = "SK"

  attribute {
    name = "PK"
    type = "S"
  }

  attribute {
    name = "SK"
    type = "S"
  }

  attribute {
    name = "GSI1PK"
    type = "S"
  }

  attribute {
    name = "GSI1SK"
    type = "S"
  }

  global_secondary_index {
    name            = "CustomerAllowlistIndex"
    hash_key        = "GSI1PK"
    range_key       = "GSI1SK"
    projection_type = "KEYS_ONLY"
  }

  ttl {
    attribute_name = "expiresAt"
    enabled        = true
  }

  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Environment = "production"
    Application = "allowlist-service"
  }
}

# DAX Cluster
resource "aws_dax_subnet_group" "allowlists" {
  name       = "allowlists-dax-subnet"
  subnet_ids = var.private_subnet_ids
}

resource "aws_dax_cluster" "allowlists" {
  cluster_name       = "allowlists-cache"
  iam_role_arn       = aws_iam_role.dax.arn
  node_type          = "dax.r5.large"
  replication_factor = 3

  subnet_group_name  = aws_dax_subnet_group.allowlists.name
  security_group_ids = [aws_security_group.dax.id]

  parameter_group_name = "default.dax1.0"

  tags = {
    Environment = "production"
  }
}

# Lambda para processar DynamoDB Streams
resource "aws_lambda_function" "stream_processor" {
  filename      = "stream_processor.zip"
  function_name = "allowlist-stream-processor"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  timeout       = 60

  environment {
    variables = {
      INVALIDATION_TOPIC = aws_sns_topic.cache_invalidation.arn
    }
  }
}

resource "aws_lambda_event_source_mapping" "dynamodb_stream" {
  event_source_arn  = aws_dynamodb_table.allowlists.stream_arn
  function_name     = aws_lambda_function.stream_processor.arn
  starting_position = "LATEST"
  batch_size        = 100
}

# SNS Topic para invalidação de cache
resource "aws_sns_topic" "cache_invalidation" {
  name = "allowlist-cache-invalidation"
}

# SQS Queue para cada pod (via subscription)
resource "aws_sqs_queue" "pod_invalidation" {
  name                       = "allowlist-pod-invalidation"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 20
}

resource "aws_sns_topic_subscription" "pod_queue" {
  topic_arn = aws_sns_topic.cache_invalidation.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.pod_invalidation.arn
}

# IAM Roles
resource "aws_iam_role" "dax" {
  name = "allowlist-dax-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "dax.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "dax_dynamodb" {
  role       = aws_iam_role.dax.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}
```

### CloudFormation (YAML)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Allowlist Service Infrastructure'

Resources:
  AllowlistsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: Allowlists
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
        - AttributeName: GSI1PK
          AttributeType: S
        - AttributeName: GSI1SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE
      GlobalSecondaryIndexes:
        - IndexName: CustomerAllowlistIndex
          KeySchema:
            - AttributeName: GSI1PK
              KeyType: HASH
            - AttributeName: GSI1SK
              KeyType: RANGE
          Projection:
            ProjectionType: KEYS_ONLY
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
      TimeToLiveSpecification:
        AttributeName: expiresAt
        Enabled: true
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true
      Tags:
        - Key: Environment
          Value: production

  DAXSubnetGroup:
    Type: AWS::DAX::SubnetGroup
    Properties:
      SubnetGroupName: allowlists-dax-subnet
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
        - !Ref PrivateSubnet3

  DAXCluster:
    Type: AWS::DAX::Cluster
    Properties:
      ClusterName: allowlists-cache
      NodeType: dax.r5.large
      ReplicationFactor: 3
      IAMRoleARN: !GetAtt DAXRole.Arn
      SubnetGroupName: !Ref DAXSubnetGroup
      SecurityGroupIds:
        - !Ref DAXSecurityGroup

  StreamProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: allowlist-stream-processor
      Runtime: nodejs18.x
      Handler: index.handler
      Role: !GetAtt LambdaRole.Arn
      Timeout: 60
      Environment:
        Variables:
          INVALIDATION_TOPIC: !Ref CacheInvalidationTopic
      Code:
        ZipFile: |
          // Lambda code here

  StreamEventSource:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt AllowlistsTable.StreamArn
      FunctionName: !Ref StreamProcessorFunction
      StartingPosition: LATEST
      BatchSize: 100

  CacheInvalidationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: allowlist-cache-invalidation

  PodInvalidationQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: allowlist-pod-invalidation
      VisibilityTimeout: 30
      MessageRetentionPeriod: 86400
      ReceiveMessageWaitTimeSeconds: 20

  QueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref CacheInvalidationTopic
      Protocol: sqs
      Endpoint: !GetAtt PodInvalidationQueue.Arn

Outputs:
  TableName:
    Value: !Ref AllowlistsTable
  DAXEndpoint:
    Value: !GetAtt DAXCluster.ClusterDiscoveryEndpoint
  InvalidationQueueUrl:
    Value: !Ref PodInvalidationQueue
```

## Troubleshooting e Debugging

### Logs Estruturados

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'allowlist.log' })
  ]
});

// Uso
logger.info('Allowlist check', {
  customerId: '550e8400-...',
  allowlistId: '123',
  type: 'FIXA',
  cacheHit: false,
  bloomFiltered: true,
  latencyMs: 2.5
});
```

### Health Check Endpoint

```javascript
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    cache: cache.getStats(),
    dynamodb: await checkDynamoDBHealth(),
    dax: await checkDAXHealth()
  };

  res.json(health);
});

async function checkDynamoDBHealth() {
  try {
    await dynamodb.describeTable({ TableName: 'Allowlists' });
    return { status: 'healthy' };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

async function checkDAXHealth() {
  try {
    await dax.getItem({
      TableName: 'Allowlists',
      Key: { PK: 'HEALTH_CHECK', SK: 'PING' }
    });
    return { status: 'healthy' };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}
```

## Checklist de Implementação

- [ ] Criar tabela DynamoDB com streams habilitado
- [ ] Criar GSI CustomerAllowlistIndex
- [ ] Provisionar DAX cluster (3 nodes)
- [ ] Implementar cache local nos pods (NodeCache)
- [ ] Implementar Bloom Filter generation (Lambda agendado)
- [ ] Configurar DynamoDB Streams → Lambda → SNS
- [ ] Criar SQS queue para cada pod receber invalidações
- [ ] Implementar sincronização de cache via SQS
- [ ] Adicionar warmup no startup dos pods
- [ ] Configurar métricas no CloudWatch
- [ ] Implementar health checks
- [ ] Configurar alertas (cache hit rate < 70%, latency p95 > 10ms)
- [ ] Testar carga (5k RPS)
- [ ] Documentar runbooks operacionais
- [ ] Configurar backup e recovery
- [ ] Implementar feature flags para rollout gradual

## Referências

- [DynamoDB Single Table Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-modeling-nosql.html)
- [DAX Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [Bloom Filter Library (JS)](https://github.com/Callidon/bloom-filters)
- [DynamoDB Streams Processing](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
