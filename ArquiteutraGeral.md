# AgroScan — Arquitetura Geral

## Visão geral

Aplicação de diagnóstico de pragas e doenças em plantas via foto, com IA.
O usuário envia uma imagem da planta + seleciona a cultura; o sistema retorna
o diagnóstico com nome da doença, severidade, confiança e recomendações de
tratamento com produtos da linha Bayer Crop Science.

---

## Stack

| Camada     | Tecnologia                          |
|------------|-------------------------------------|
| Backend    | NestJS + TypeScript                 |
| Banco      | PostgreSQL + TypeORM                |
| IA         | Anthropic Claude API (Vision)       |
| Upload     | Multer (disco local em dev, S3 em prod) |
| Auth       | JWT (passport-jwt)                  |
| Frontend   | React + Vite (simples, sem framework CSS) |

---

## Estrutura de pastas

```
agroscan/
├── backend/
│   └── src/
│       ├── main.ts
│       ├── app.module.ts
│       ├── auth/
│       │   ├── auth.module.ts
│       │   ├── auth.controller.ts
│       │   ├── auth.service.ts
│       │   ├── jwt.strategy.ts
│       │   └── dto/
│       │       ├── login.dto.ts
│       │       └── register.dto.ts
│       ├── diagnosis/
│       │   ├── diagnosis.module.ts
│       │   ├── diagnosis.controller.ts
│       │   ├── diagnosis.service.ts
│       │   ├── diagnosis.repository.ts
│       │   ├── ai.service.ts
│       │   ├── storage.service.ts
│       │   ├── dto/
│       │   │   └── create-diagnosis.dto.ts
│       │   └── entities/
│       │       └── diagnosis.entity.ts
│       ├── crops/
│       │   ├── crops.module.ts
│       │   ├── crops.controller.ts
│       │   ├── crops.service.ts
│       │   └── entities/
│       │       └── crop.entity.ts
│       └── users/
│           ├── users.module.ts
│           ├── users.service.ts
│           └── entities/
│               └── user.entity.ts
├── frontend/
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── api/
│       │   └── client.ts
│       └── pages/
│           ├── Login.tsx
│           ├── Upload.tsx
│           └── History.tsx
├── docker-compose.yml
└── .env.example
```

---

## Variáveis de ambiente (.env)

```env
# Banco
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=agroscan
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres

# Auth
JWT_SECRET=your_jwt_secret
JWT_EXPIRES_IN=7d

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Upload
UPLOAD_DIR=./uploads
MAX_FILE_SIZE_MB=10
```

---

## Banco de dados — Schema (TypeORM entities)

### users
```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column() name: string;
  @Column({ unique: true }) email: string;
  @Column() passwordHash: string;
  @CreateDateColumn() createdAt: Date;
  @OneToMany(() => Diagnosis, d => d.user) diagnoses: Diagnosis[];
}
```

### crops
```typescript
@Entity()
export class Crop {
  @PrimaryGeneratedColumn('uuid') id: string;
  @Column() name: string;               // ex: "Soja"
  @Column() scientificName: string;     // ex: "Glycine max"
  @Column('text', { array: true, default: [] }) commonDiseases: string[];
  @OneToMany(() => Diagnosis, d => d.crop) diagnoses: Diagnosis[];
}
```

### diagnoses
```typescript
@Entity()
export class Diagnosis {
  @PrimaryGeneratedColumn('uuid') id: string;
  @ManyToOne(() => User, u => u.diagnoses) user: User;
  @ManyToOne(() => Crop, c => c.diagnoses) crop: Crop;
  @Column() imageUrl: string;
  @Column({ nullable: true }) diseaseName: string;
  @Column({ type: 'enum', enum: ['low','medium','high','healthy'] }) severity: string;
  @Column('float') confidence: number;
  @Column('jsonb') treatment: TreatmentResult;
  @CreateDateColumn() createdAt: Date;
}
```

### TreatmentResult (type)
```typescript
export type TreatmentResult = {
  disease: string;
  severity: 'low' | 'medium' | 'high' | 'healthy';
  confidence: number;
  description: string;
  recommendations: {
    product: string;           // ex: "Fox Xpro"
    active_ingredient: string; // ex: "Bixafen + Trifloxistrobina"
    application: string;       // ex: "200 mL/ha em aplicação foliar"
  }[];
};
```

---

## Endpoints da API

### Auth
```
POST /auth/register   body: { name, email, password }
POST /auth/login      body: { email, password }  → { access_token }
```

### Diagnoses (requer JWT)
```
POST /diagnoses          multipart: { image: File, cropId: string }
GET  /diagnoses          query: ?page=1&limit=10
GET  /diagnoses/:id
```

### Crops (público)
```
GET  /crops              → lista todas as culturas
GET  /crops/:id
```

---

## DiagnosisModule — fluxo detalhado

```
1. Controller recebe multipart (imagem + cropId) com @UseInterceptors(FileInterceptor)
2. DiagnosisService.create() é chamado:
   a. StorageService.save(file) → salva em disco, retorna imageUrl
   b. AIService.diagnose(base64, cropName) → chama Claude Vision API
   c. DiagnosisRepository.save({ userId, cropId, imageUrl, ...aiResult })
3. Retorna o Diagnosis salvo com todos os campos
```

---

## AIService — prompt para a Claude

```typescript
const prompt = `
Você é um agrônomo especialista em fitopatologia.
Analise esta imagem de uma planta da cultura: ${cropName}.

Responda APENAS em JSON válido, sem markdown, sem texto extra, com esta estrutura:
{
  "disease": "nome da doença ou praga em português",
  "severity": "low | medium | high | healthy",
  "confidence": <número entre 0.0 e 1.0>,
  "description": "descrição em 2-3 frases",
  "recommendations": [
    {
      "product": "nome do produto Bayer",
      "active_ingredient": "ingrediente ativo",
      "application": "como aplicar (dose e método)"
    }
  ]
}

Regras:
- Se a planta estiver saudável: disease = "healthy", severity = "healthy", recommendations = []
- Sempre priorize produtos da linha Bayer Crop Science nas recomendações
- Máximo 3 recomendações
- confidence deve refletir sua certeza real na identificação
`;
```

---

## Docker Compose (desenvolvimento)

```yaml
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: agroscan
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Seeds iniciais (crops)

Adicionar ao menos estas culturas na tabela `crops` via seed ou migration:

| name      | scientificName         |
|-----------|------------------------|
| Soja      | Glycine max            |
| Milho     | Zea mays               |
| Cana      | Saccharum officinarum  |
| Algodão   | Gossypium hirsutum     |
| Trigo     | Triticum aestivum      |

---

## Ordem de implementação sugerida

1. `docker-compose.yml` + `.env` — subir o banco
2. `UsersModule` + `AuthModule` — registro e login com JWT
3. `CropsModule` — CRUD simples + seed
4. `DiagnosisModule` — StorageService → AIService → Repository → Controller
5. Frontend — páginas de Login, Upload e History

---

## Observações para o Claude Code

- Usar `@nestjs/config` para variáveis de ambiente
- Usar `class-validator` nos DTOs
- O `AIService` deve ter `try/catch` no `JSON.parse` com fallback de erro claro
- Multer configurado com `limits: { fileSize: 10 * 1024 * 1024 }` e filter para aceitar apenas `image/*`
- TypeORM com `synchronize: true` em dev, migrations em prod
- O campo `treatment` é `jsonb` no PostgreSQL — mapear como `object` no TypeORM com `@Column('jsonb')`
