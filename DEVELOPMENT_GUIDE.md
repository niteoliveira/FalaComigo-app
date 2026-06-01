# 🛠️ Guia de Desenvolvimento Iterativo — Reconstrói Jacareí

> **Objetivo:** Transformar o app de um estado funcional com dados reais (Firebase Auth + Firestore)
> em um produto completo com as regras de negócio v3 (Score, Clusterização, Reputação, Foto Obrigatória).
>
> **Princípio:** Cada fase entrega algo **testável e funcional**. Nunca quebre o que já funciona.

---

## 📐 Princípios de Desenvolvimento

### 1. TDD — Test-Driven Development

Cada funcionalidade segue o ciclo **Red → Green → Refactor**:

```
1. RED    — Escreva o teste ANTES do código. Ele DEVE falhar.
2. GREEN  — Escreva o MÍNIMO de código para o teste passar.
3. REFACTOR — Limpe o código sem quebrar o teste.
```

**Regras práticas:**
- Nunca escreva código de produção sem um teste falhando primeiro.
- Testes de unidade para regras de negócio (providers, models, repository).
- Widget tests para fluxos de UI críticos.
- Nomeie os testes descritivamente: `'confirmProblem deve incrementar score quando usuário está a menos de 200m'`.

### 2. Commits Atômicos

```
feat(score): implementar cálculo de score com peso por distância
test(score): adicionar testes para ScoreCalculator
fix(quickconfirm): corrigir dismiss chamando denyProblem
refactor(models): extrair ProblemStatus.pending e expired
```

- Um commit = uma mudança lógica.
- Sempre rode `flutter test` antes de commitar.

### 3. Arquitetura Limpa

```
lib/
├── core/           # Theme, router, constantes — NÃO depende de nada
├── models/         # Modelos de dados puros — NÃO dependem de Flutter
├── data/
│   └── repositories/  # Abstração + implementação Firestore
├── services/       # Serviços (geocoding, camera, geolocation)
├── providers/      # Riverpod — cola entre UI e dados
├── widgets/        # Componentes reutilizáveis
└── screens/        # Telas compostas de widgets
```

**Regra de ouro:** Dependências apontam para DENTRO (screens → providers → repositories → models).
Nunca ao contrário.

### 4. Convenções

| Item | Convenção |
|------|-----------|
| Nomes de arquivos | `snake_case.dart` |
| Classes | `PascalCase` |
| Variáveis/métodos | `camelCase` |
| Constantes | `camelCase` ou `SCREAMING_SNAKE` para valores globais |
| Testes | `nome_do_arquivo_test.dart` em `/test` espelhando `/lib` |
| Strings de UI | Centralizadas em `app_strings.dart` |
| Cores | Centralizadas em `app_colors.dart` |

---

## 🗺️ Diagnóstico: Estado Atual vs. Estado Desejado

### O que JÁ funciona ✅

| Componente | Status |
|------------|--------|
| Firebase Auth (email/senha + registro) | ✅ Completo |
| Guard de rota (GoRouter + redirect) | ✅ Completo |
| LoginScreen | ✅ Completo |
| Firestore CRUD (add, update, delete, stream) | ✅ Completo |
| ProblemsRepository (abstração + implementação) | ✅ Completo |
| QuickConfirmCard com filtro RN-C2 | ✅ Completo |
| Mapa real (flutter_map + CartoDB) | ✅ Completo |
| Filtros (status/tipo/gravidade) | ✅ Completo |
| Tela de perfil com stats do Firestore | ✅ Completo |
| 7 arquivos de teste existentes | ✅ Completo |

### O que PRECISA mudar 🔄

| Gap | Impacto | Fase |
|-----|---------|------|
| Status `pending`, `contested`, `expired` não existem no enum | 🔥 Bloqueante | 1 |
| Campo `score` não existe no modelo | 🔥 Bloqueante | 1 |
| Foto é opcional — precisa ser obrigatória | 🔥 Core | 2 |
| Sistema de Score não implementado | 🔥 Core | 2 |
| `onDismiss` do QuickConfirmCard chama `denyProblem` (bug) | 🐛 Bug | 1 |
| Decay semanal não existe | ✨ Melhoria | 3 |
| Reputação de usuário não existe | ✨ Melhoria | 3 |
| Clusterização (merge + visual) não existe | ✨ Melhoria | 4 |
| Geolocalização real (geolocator) | ✨ Melhoria | 4 |
| Push notifications por proximidade | ✨ Futuro | 5 |
| `testRead()`/`testWrite()` no `initState` | 🔥 Lixo | 0 |

---

## 🚀 Fases de Desenvolvimento

---

### Fase 0 — Limpeza e Estabilização (1-2 horas)

> **Meta:** Remover código de teste esquecido e estabilizar a base.

#### Tarefas

- [x] **0.1** Remover `testRead()` e `testWrite()` do `home_screen.dart`
- [x] **0.2** Remover chamadas no `initState()`
- [x] **0.3** Corrigir `onDismiss` do QuickConfirmCard — criar estado local de `dismissedIds` em vez de chamar `denyProblem`
- [x] **0.4** Rodar `flutter analyze` e resolver todos os warnings
- [x] **0.5** Rodar `flutter test` e garantir que todos os 7 testes passam

#### TDD para 0.3

```dart
// test/quick_confirm_dismiss_test.dart
// RED: O teste falha porque dismiss ainda chama denyProblem
test('dismiss do QuickConfirmCard NÃO deve chamar denyProblem', () {
  // Arrange: criar HomeScreen com mock provider
  // Act: disparar onDismiss
  // Assert: verify que denyProblem NÃO foi chamado
  //         verify que o card sumiu da tela
});

test('dismiss do QuickConfirmCard deve ser temporário (volta ao reabrir app)', () {
  // Arrange: dismiss um card
  // Act: simular rebuild do widget
  // Assert: card volta a aparecer
});
```

#### Critério de Aceite

```bash
flutter analyze   # 0 issues
flutter test      # Todos passam (7 existentes + novos)
```

---

### Fase 1 — Expandir o Modelo de Dados (2-3 horas)

> **Meta:** Atualizar `ProblemReport`, `ProblemStatus` e o repositório para suportar as regras v3.

#### 1.1 Expandir `ProblemStatus`

```dart
enum ProblemStatus {
  pending,    // NOVO — recém criado
  active,     // Score ≥ 2
  analysis,   // Score ≥ 10
  contested,  // NOVO — votos conflitantes
  resolved,
  expired,    // NOVO — morreu por abandono
}
```

#### 1.2 Adicionar campos ao `ProblemReport`

```dart
// Campos novos necessários:
final double score;                 // Score calculado
final DateTime lastInteraction;     // Para decay
final String reportedByUserId;      // UID puro do criador
final List<String> photoUrls;       // PLURAL — substituir imageUrl
final List<String> confirmedByIds;  // Quem confirmou
final List<String> deniedByIds;     // Quem negou
```

#### 1.3 Atualizar `AppColors` com novas cores

```dart
static const Color statusPending = Color(0xFF9E9E9E);   // Cinza
static const Color statusContested = Color(0xFFFF6D00);  // Laranja
```

#### TDD

```dart
// test/models/problem_report_test.dart

group('ProblemStatus expandido', () {
  test('pending deve ter label "Pendente"', () {
    expect(ProblemStatus.pending.label, 'Pendente');
  });

  test('contested deve ter cor laranja', () {
    expect(ProblemStatus.contested.color, AppColors.statusContested);
  });

  test('expired deve ter label "Expirado"', () {
    expect(ProblemStatus.expired.label, 'Expirado');
  });
});

group('ProblemReport.fromJson com campos novos', () {
  test('deve parsear score e lastInteraction do JSON', () {
    final json = { /* ... com score: 3.5, lastInteraction: '...' */ };
    final report = ProblemReport.fromJson(json);
    expect(report.score, 3.5);
    expect(report.lastInteraction, isNotNull);
  });

  test('deve manter retrocompatibilidade com JSON sem score', () {
    final jsonAntigo = { /* JSON sem campo score */ };
    final report = ProblemReport.fromJson(jsonAntigo);
    expect(report.score, 1.0); // default
  });
});
```

#### Checklist de Migração do Firestore

- [x] Documentos existentes na coleção `reports` não têm `score` → `fromJson` deve ter default `1.0`
- [x] `imageUrl` (singular) deve migrar para `photoUrls` (lista) → tratar ambos no `fromJson`
- [x] `status: 'active'` nos docs existentes continua funcionando
- [x] Novos docs devem nascer com `status: 'pending'`

#### Critério de Aceite

```bash
flutter test   # Todos passam incluindo testes novos de modelo
flutter run    # App abre sem crashar com dados existentes no Firestore
```

---

### Fase 2 — Sistema de Score (3-4 horas)

> **Meta:** Implementar o cálculo de Score e as transições automáticas de status.

#### 2.1 Criar `ScoreCalculator` (lógica pura, sem Firebase)

```
lib/
└── core/
    └── score/
        └── score_calculator.dart
```

```dart
/// Calcula o score de um report baseado nos votos.
/// Classe PURA — sem dependências externas, 100% testável.
class ScoreCalculator {
  static const double initialScore = 1.0;
  static const double weeklyDecay = -0.5;
  static const double activationThreshold = 2.0;
  static const double analysisThreshold = 10.0;
  static const double resolvedThreshold = -3.0;

  /// Calcula o multiplicador de distância para um voto
  static double distanceMultiplier(double distanceMeters) {
    if (distanceMeters <= 200) return 2.0;
    if (distanceMeters <= 500) return 1.5;
    return 1.0;
  }

  /// Calcula o score total de um report
  static double calculateScore({
    required List<Vote> votes,
    required DateTime createdAt,
    required DateTime now,
  }) { /* ... */ }

  /// Determina o status correto baseado no score
  static ProblemStatus determineStatus({
    required double score,
    required ProblemStatus currentStatus,
    required DateTime? scoreNegativeSince,
    required DateTime now,
  }) { /* ... */ }
}
```

#### 2.2 Atualizar `ProblemsRepository` para calcular score

- `confirmProblem()` → registrar voto com distância → recalcular score → atualizar status se necessário
- `denyProblem()` → registrar voto negativo → recalcular score

#### 2.3 Atualizar `ReportProblemSheet` para criar com status `pending`

- Problema novo nasce com `status: pending`, `score: 1.0`

#### TDD — ScoreCalculator

```dart
// test/core/score/score_calculator_test.dart

group('ScoreCalculator', () {
  group('distanceMultiplier', () {
    test('≤ 200m deve retornar 2.0', () {
      expect(ScoreCalculator.distanceMultiplier(150), 2.0);
    });

    test('200-500m deve retornar 1.5', () {
      expect(ScoreCalculator.distanceMultiplier(350), 1.5);
    });

    test('> 500m deve retornar 1.0', () {
      expect(ScoreCalculator.distanceMultiplier(1000), 1.0);
    });
  });

  group('calculateScore', () {
    test('report novo sem votos deve ter score = 1.0', () {
      final score = ScoreCalculator.calculateScore(
        votes: [],
        createdAt: DateTime.now(),
        now: DateTime.now(),
      );
      expect(score, 1.0);
    });

    test('1 voto positivo próximo deve ativar (score ≥ 2)', () {
      final votes = [
        Vote(type: VoteType.confirm, distanceMeters: 150, userLevel: 1),
      ];
      final score = ScoreCalculator.calculateScore(
        votes: votes,
        createdAt: DateTime.now(),
        now: DateTime.now(),
      );
      expect(score, greaterThanOrEqualTo(2.0)); // 1 + (1 × 1 × 2.0) = 3.0
    });

    test('decay de 2 semanas sem interação deve reduzir score em 1.0', () {
      final twoWeeksAgo = DateTime.now().subtract(Duration(days: 14));
      final score = ScoreCalculator.calculateScore(
        votes: [],
        createdAt: twoWeeksAgo,
        now: DateTime.now(),
      );
      expect(score, 0.0); // 1.0 - (2 × 0.5) = 0.0
    });
  });

  group('determineStatus', () {
    test('score ≥ 2 com status pending deve transicionar para active', () {
      final status = ScoreCalculator.determineStatus(
        score: 2.5,
        currentStatus: ProblemStatus.pending,
        scoreNegativeSince: null,
        now: DateTime.now(),
      );
      expect(status, ProblemStatus.active);
    });

    test('score ≤ -3 por 3+ dias deve transicionar para resolved', () {
      final threeDaysAgo = DateTime.now().subtract(Duration(days: 4));
      final status = ScoreCalculator.determineStatus(
        score: -3.5,
        currentStatus: ProblemStatus.active,
        scoreNegativeSince: threeDaysAgo,
        now: DateTime.now(),
      );
      expect(status, ProblemStatus.resolved);
    });
  });
});
```

#### Critério de Aceite

- `ScoreCalculator` tem 100% dos cenários da tabela v3 cobertos por testes.
- Confirmar um problema no app recalcula score e muda status automaticamente.
- Report criado pelo app nasce como `Pendente` (cinza no mapa).

---

### Fase 3 — Foto Obrigatória + Reputação (3-4 horas)

> **Meta:** Implementar câmera obrigatória no reporte e sistema de reputação invisível.

#### 3.1 Foto Obrigatória no Report

- [x] Adicionar dependência `image_picker` ao `pubspec.yaml`
- [x] Criar `CameraService` em `lib/services/camera_service.dart`
- [x] Refatorar `ReportProblemSheet`: câmera abre PRIMEIRO, antes do formulário
- [ ] Implementar upload para Firebase Storage (ou gerar URL temporária)
- [x] Botão "Confirmar" desabilitado se `photoUrls` estiver vazia

#### 3.2 Sistema de Reputação

```
lib/
└── core/
    └── reputation/
        └── reputation_calculator.dart
```

```dart
/// Calcula o nível de reputação do usuário.
/// Classe PURA, 100% testável.
class ReputationCalculator {
  /// Retorna o nível (1, 2 ou 3) baseado nas estatísticas do usuário
  static int calculateLevel({
    required int reportsWithScoreAbove2,
    required int totalVotesGiven,
    required DateTime accountCreatedAt,
    required DateTime now,
  }) {
    final accountAgeDays = now.difference(accountCreatedAt).inDays;

    if (reportsWithScoreAbove2 >= 20 && totalVotesGiven >= 50 && accountAgeDays >= 90) {
      return 3; // Guardião
    }
    if (reportsWithScoreAbove2 >= 5 && totalVotesGiven >= 20) {
      return 2; // Confiável
    }
    return 1; // Novo
  }
}
```

#### 3.3 Atualizar Firestore `users` collection

Campos novos no documento do usuário:
```json
{
  "reportsWithScoreAbove2": 0,
  "totalVotesGiven": 0,
  "createdAt": "...",
  "reputationLevel": 1
}
```

#### TDD — ReputationCalculator

```dart
// test/core/reputation/reputation_calculator_test.dart

group('ReputationCalculator', () {
  test('conta nova deve ser nível 1', () {
    expect(ReputationCalculator.calculateLevel(
      reportsWithScoreAbove2: 0,
      totalVotesGiven: 0,
      accountCreatedAt: DateTime.now(),
      now: DateTime.now(),
    ), 1);
  });

  test('5+ reports validados e 20+ votos deve ser nível 2', () {
    expect(ReputationCalculator.calculateLevel(
      reportsWithScoreAbove2: 5,
      totalVotesGiven: 20,
      accountCreatedAt: DateTime.now().subtract(Duration(days: 30)),
      now: DateTime.now(),
    ), 2);
  });

  test('20+ reports, 50+ votos e 90+ dias deve ser nível 3', () {
    expect(ReputationCalculator.calculateLevel(
      reportsWithScoreAbove2: 20,
      totalVotesGiven: 50,
      accountCreatedAt: DateTime.now().subtract(Duration(days: 100)),
      now: DateTime.now(),
    ), 3);
  });
});
```

#### Critério de Aceite

- Criar report sem foto é impossível (botão desabilitado + validação).
- `ReputationCalculator` com 100% de cobertura.
- Score de voto usa o nível de reputação do votante como multiplicador.

---

### Fase 4 — Clusterização + Geolocalização (4-5 horas)

> **Meta:** Evitar poluição visual no mapa e usar GPS real.

#### 4.1 Clusterização Visual (Camada 2 — mais fácil, fazer primeiro)

- [x] Adicionar dependência `flutter_map_marker_cluster` ao `pubspec.yaml`
- [x] Refatorar `MapView` para usar `MarkerClusterLayerOptions`
- [x] Cluster exibe cor do status mais grave do grupo
- [x] Badge numérico no cluster
- [x] Tocar no cluster dá zoom até separar os pinos

#### 4.2 Sugestão de Merge na Criação (Camada 1)

- [x] Ao abrir `ReportProblemSheet`, fazer query de reports do mesmo tipo em raio de 100m
- [x] Se encontrar, exibir dialog: "Já existe um report aqui. É o mesmo?"
- [x] Se "Sim" → converter ação em confirmação + foto adicional
- [x] Se "Não" → criar novo report normalmente

**Nota sobre Geohash:** Firestore não tem query geoespacial nativa.
Opções:
1. **GeoFlutterFire** — pacote que adiciona geohash ao Firestore
2. **Query client-side** — carregar reports ativos e filtrar por distância no Dart (aceitável com poucos dados)
3. **Cloud Function** — calcular no backend

**Recomendação para MVP:** Opção 2 (client-side). Com poucos reports iniciais, a performance é aceitável. Migrar para geohash quando escalar.

#### 4.3 Geolocalização Real

- [x] Adicionar dependência `geolocator` e `geocoding` ao `pubspec.yaml` (geolocator já presente)
- [ ] Criar `LocationService` em `lib/services/location_service.dart`
- [x] FAB de localização usa GPS real em vez de centralizar em Jacareí hardcoded
- [x] Report usa coordenadas GPS reais (não mais pino arrastável como opção default)
- [x] Atualizar `GeocodingService` para usar API real (Nominatim gratuito)

#### TDD — Merge Detection

```dart
// test/core/clustering/merge_detector_test.dart

group('MergeDetector', () {
  test('deve sugerir merge para report do mesmo tipo a 50m', () {
    final existing = [mockReport(type: ProblemType.hole, lat: -23.305, lng: -45.965)];
    final newReport = mockReport(type: ProblemType.hole, lat: -23.3054, lng: -45.965);

    final suggestion = MergeDetector.findNearbyReports(existing, newReport, radiusMeters: 100);
    expect(suggestion, isNotEmpty);
  });

  test('NÃO deve sugerir merge para tipo diferente a 50m', () {
    final existing = [mockReport(type: ProblemType.hole, lat: -23.305, lng: -45.965)];
    final newReport = mockReport(type: ProblemType.crack, lat: -23.3054, lng: -45.965);

    final suggestion = MergeDetector.findNearbyReports(existing, newReport, radiusMeters: 100);
    expect(suggestion, isEmpty);
  });

  test('NÃO deve sugerir merge para report a 200m', () {
    final existing = [mockReport(type: ProblemType.hole, lat: -23.305, lng: -45.965)];
    final newReport = mockReport(type: ProblemType.hole, lat: -23.307, lng: -45.967);

    final suggestion = MergeDetector.findNearbyReports(existing, newReport, radiusMeters: 100);
    expect(suggestion, isEmpty);
  });
});
```

#### Critério de Aceite

- Mapa agrupa pinos próximos em zoom baixo.
- Ao criar report perto de outro do mesmo tipo, app sugere merge.
- GPS real funciona para localização do usuário.

---

### Fase 5 — Decay, Anti-Abuso e Polimento (3-4 horas)

> **Meta:** Implementar decay automático, regras anti-abuso e polimento final.

#### 5.1 Decay Automático

**Opção A — Cloud Function (ideal):**
```
Cloud Function schedulada (cron) roda 1x/dia:
  - Para cada report com lastInteraction > 7 dias:
    - Aplicar decay de -0.5/semana no score
    - Recalcular status
    - Se Pendente + 7 dias sem votos → Expirado
```

**Opção B — Client-side (MVP):**
```
Ao carregar um report, calcular decay baseado em:
  score_efetivo = score_armazenado - (semanas_sem_interação × 0.5)
```

**Recomendação:** Opção B para MVP, migrar para A quando tiver Cloud Functions.

#### 5.2 Regras Anti-Abuso

- [x] **RA-3:** Conta nova com < 24h não pode negar reports (pode confirmar)
- [x] Verificar no `ProblemsRepository.ignoreProblem()` a data de criação da conta
- [x] Limites diários: 5 reports/dia, 20 votos/dia

#### 5.3 Filtros Expandidos

- [x] Adicionar filtros "Resolvidos" e "Expirados" no `FilterSheet`
- [x] Por padrão, NÃO exibir resolvidos/expirados
- [x] Quando filtro de resolvidos ativo, mostrar pinos verdes

#### 5.4 Polimento de UX

- [x] Feedback visual ao votar (SnackBar com ícone ao confirmar e negar)
- [ ] Push notification ao criador quando report ativa: "Seu report foi validado! 🎉"
- [ ] Contagem de reports do usuário na tela de perfil separada por status
- [x] Loading states em todas as ações assíncronas (confirm, deny, report)

#### Critério de Aceite

- Reports sem interação por 7+ dias perdem score gradualmente.
- Conta nova não consegue negar nos primeiros 24h.
- Filtro "Resolvidos" mostra/esconde pinos verdes.

---

## 📋 Checklist de Qualidade por Fase

Antes de considerar uma fase **concluída**, verificar:

```
[ ] flutter analyze — 0 issues
[ ] flutter test — 100% passando
[ ] flutter run — app abre e funciona no emulador
[ ] Dados existentes no Firestore continuam funcionando (retrocompatibilidade)
[ ] Código novo tem testes
[ ] Nenhum print() ou debugPrint() em código de produção
[ ] Nenhum TODO sem issue/tarefa associada
[ ] Commit atômico com mensagem descritiva
```

---

## 🧪 Estrutura de Testes Planejada

```
test/
├── core/
│   ├── score/
│   │   └── score_calculator_test.dart          # Fase 2
│   ├── reputation/
│   │   └── reputation_calculator_test.dart     # Fase 3
│   └── clustering/
│       └── merge_detector_test.dart            # Fase 4
├── models/
│   └── problem_report_test.dart                # Fase 1
├── providers/
│   └── providers_test.dart                     # Existente + expansão
├── data/
│   └── mock_problems_repository.dart           # Existente
├── widgets/
│   └── quick_confirm_dismiss_test.dart         # Fase 0
├── home_screen_test.dart                       # Existente
├── profile_screen_test.dart                    # Existente
├── search_screen_test.dart                     # Existente
└── theme_test.dart                             # Existente
```

**Meta de cobertura:** ≥ 80% nas classes de lógica (`ScoreCalculator`, `ReputationCalculator`, `MergeDetector`, `ProblemsNotifier`).

---

## 📦 Dependências a Adicionar por Fase

| Fase | Pacote | Motivo |
|------|--------|--------|
| 3 | `image_picker` | Câmera para foto obrigatória |
| 3 | `firebase_storage` | Upload de fotos |
| 4 | `flutter_map_marker_cluster` | Agrupamento visual de pinos |
| 4 | `geolocator` | GPS real do dispositivo |
| 4 | `geocoding` | Endereço reverso real |
| 5 | `firebase_messaging` *(futuro)* | Push notifications |

---

## ⏱️ Estimativa de Tempo Total

| Fase | Tempo Estimado | Acumulado |
|------|---------------|-----------|
| **Fase 0** — Limpeza | 1-2h | 2h |
| **Fase 1** — Modelo de Dados | 2-3h | 5h |
| **Fase 2** — Sistema de Score | 3-4h | 9h |
| **Fase 3** — Foto + Reputação | 3-4h | 13h |
| **Fase 4** — Cluster + GPS | 4-5h | 18h |
| **Fase 5** — Decay + Anti-Abuso | 3-4h | 22h |

**Total estimado:** ~20-22 horas de desenvolvimento focado.

---

## 🔄 Como Usar Este Documento

1. **Escolha a fase atual** — siga em ordem (0 → 1 → 2 → ...).
2. **Leia as tarefas e os testes primeiro** — entenda O QUE antes do COMO.
3. **Escreva o teste (RED)** — copie o esqueleto de teste da fase, adapte e rode. Deve falhar.
4. **Implemente o mínimo (GREEN)** — faça o teste passar com o código mais simples.
5. **Refatore (REFACTOR)** — limpe, extraia, renomeie. Os testes garantem que não quebrou.
6. **Marque o checkbox** — atualize este documento conforme avança.
7. **Commite** — um commit atômico por tarefa concluída.
8. **Rode o checklist de qualidade** — antes de avançar para a próxima fase.

> [!TIP]
> **Dica:** Se uma tarefa parece grande demais, quebre em subtarefas menores.
> A regra é: se demorou mais de 45 minutos sem commitar, você está fazendo coisa demais de uma vez.
