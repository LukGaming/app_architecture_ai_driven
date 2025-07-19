# 🔄 Refatoração: Agendamentos - App Architecture (Google/Flutter)

## 📋 Visão Geral

Este documento demonstra a refatoração completa da funcionalidade de **Agendamentos** do projeto, migrando da arquitetura antiga (Clean Architecture + StateNotifier) para a nova **App Architecture** recomendada pelo Google/Flutter.

## 🏗️ Arquitetura Anterior vs Nova

### ❌ Anterior (Clean Architecture)
```
Controllers → Logic → Services → Models
    ↓
StateNotifier com estados customizados
```

### ✅ Nova (App Architecture - Google/Flutter)
```
ViewModels → UseCases → Repositories → Services
    ↓
ChangeNotifier + Commands para reatividade
```

## 📁 Estrutura dos Arquivos Refatorados

### 🛠️ Utils Layer
```
lib/utils/
├── command.dart          # ✅ Commands para operações reativas
└── result.dart           # ✅ ResultApp para tratamento de erros
```

### 💾 Data Layer
```
lib/data/
├── repositories/
│   └── agendamentos_repository.dart       # ✅ Implementação concreta
└── services/
    └── agendamentos_api_service.dart      # ✅ Serviço API
```

### 🏢 Domain Layer
```
lib/domain/
├── repositories/
│   └── agendamentos_repository.dart       # ✅ Contrato abstrato
└── usecases/
    └── agendamentos_usecases.dart         # ✅ UseCases específicos
```

### 🎨 UI Layer
```
lib/ui/agendamentos/
├── viewmodels/
│   └── agendamentos_viewmodel_new.dart    # ✅ ViewModel reativo
├── widgets/
│   ├── agendamentos_list_new.dart         # ✅ Lista reativa
│   ├── agendamento_card_new.dart          # ✅ Card melhorado
│   └── select_agendamento_day_new.dart    # ✅ Seletor de data
└── pages/
    ├── agendamentos_page_new.dart         # ✅ Página principal
    └── exemplo_agendamentos_refatorado.dart # ✅ Exemplo completo
```

## 🔧 Principais Mudanças

### 1. **Repository com ChangeNotifier**
O `AgendamentosRepository` agora estende `ChangeNotifier` para notificar mudanças automaticamente:

```dart
abstract class AgendamentosRepository extends ChangeNotifier {
  List<Agendamento> get agendamentos;
  DateTime? get dataSelecionada;
  
  Future<ResultApp<List<Agendamento>>> getAgendamentosDoDia(DateTime data);
  // ... outros métodos
}
```

### 2. **Commands para Operações Reativas**
Substituição de métodos manuais por Commands que gerenciam estado automaticamente:

```dart
class AgendamentosViewModel extends ChangeNotifier {
  // Commands instanciados diretamente
  late final getAgendamentosDoDiaCommand = Command1<List<Agendamento>, DateTime>(
    _getAgendamentosDoDia,
  );
  
  // Método privado chamado pelo Command
  Future<ResultApp<List<Agendamento>>> _getAgendamentosDoDia(DateTime data) async {
    return await _getAgendamentosDoDiaUseCase.execute(data);
  }
}
```

### 3. **UI Reativa com ListenableBuilder**
Widgets que reagem automaticamente às mudanças dos Commands:

```dart
ListenableBuilder(
  listenable: agendamentosViewModel.getAgendamentosDoDiaCommand,
  builder: (context, child) {
    final command = agendamentosViewModel.getAgendamentosDoDiaCommand;
    
    if (command.running) {
      return const CircularProgressIndicator();
    }
    
    if (command.error) {
      return Text('Erro: ${command.result?.error}');
    }
    
    // Exibir dados
    return ListView(...);
  },
)
```

### 4. **Fonte Única da Verdade**
O Repository é a única fonte de dados, o ViewModel apenas delega:

```dart
class AgendamentosViewModel extends ChangeNotifier {
  // Getters que delegam para o repository
  List<Agendamento> get agendamentos => _agendamentosRepository.agendamentos;
  DateTime? get dataSelecionada => _agendamentosRepository.dataSelecionada;
  
  // Repository notifica mudanças automaticamente via ChangeNotifier
}
```

## 🎯 Vantagens da Nova Arquitetura

### ✅ **Reatividade Automática**
- Commands gerenciam estado (loading, resultado, erro) automaticamente
- Repository notifica mudanças via ChangeNotifier
- UI reage sem código manual

### ✅ **Simplicidade**
- Menos boilerplate que StateNotifier + Estados customizados
- Commands instanciados diretamente no ViewModel
- Menos camadas e complexidade

### ✅ **Fonte Única da Verdade**
- Repository mantém o estado
- Cache automático no Repository
- Sincronização garantida entre UI e dados

### ✅ **Testabilidade**
- UseCases isolados e testáveis
- Repository mockável
- Commands testáveis individualmente

## 🚀 Como Usar

### 1. **Injeção de Dependência**
```dart
// No setupInjector()
i.registerLazySingleton<AgendamentosRepository>(
  () => AgendamentosRepositoryRemote(i(), i()),
);
i.registerLazySingleton<AgendamentosViewModel>(
  () => AgendamentosViewModel(i(), i(), i(), i()),
);
```

### 2. **Na UI**
```dart
class MinhaPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final viewModel = injector<AgendamentosViewModel>();
    
    return Scaffold(
      body: Column(
        children: [
          // Usar os Commands
          ElevatedButton(
            onPressed: () => viewModel.carregarAgendamentos(DateTime.now()),
            child: Text('Carregar'),
          ),
          
          // Reagir aos Commands
          ListenableBuilder(
            listenable: viewModel.getAgendamentosDoDiaCommand,
            builder: (context, child) {
              // UI reativa aqui
            },
          ),
        ],
      ),
    );
  }
}
```

### 3. **Exemplo Completo**
Veja o arquivo `exemplo_agendamentos_refatorado.dart` para um exemplo completo de uso.

## 🔄 Migração

### Arquivos Substituídos:
- ❌ `AgendamentosController` → ✅ `AgendamentosViewModel`
- ❌ `AgendamentoLogic` → ✅ `AgendamentosUseCases`
- ❌ `AgendamentosService` → ✅ `AgendamentosApiService`
- ❌ `StateBuilder` → ✅ `ListenableBuilder`

### Fluxo de Dados:
- ❌ `UI → Controller → Logic → Service`
- ✅ `UI → ViewModel → UseCase → Repository → Service`

## 📊 Comparação de Código

### Anterior (StateNotifier):
```dart
// Controller com muito boilerplate
class AgendamentosController extends StateNotifier<BaseState> {
  Future<void> loadAgendamentos({required DateTime date}) async {
    update(LoadingState());
    try {
      await _agendamentosLogic.listSchedulesFromTheDay(date).fold(
        (success) => update(AgendamentosLoadedState(success)),
        (error) => update(ErrorState(error.message)),
      );
    } catch (e) {
      update(ErrorState(e.toString()));
    }
  }
}

// UI com StateBuilder
StateBuilder(
  notifier: _agendamentosController,
  builder: (context, state) {
    if (state is LoadingState) return CircularProgressIndicator();
    if (state is AgendamentosLoadedState) return ListView(...);
    if (state is ErrorState) return Text(state.message);
    return SizedBox.shrink();
  },
)
```

### Nova (Commands):
```dart
// ViewModel simples
class AgendamentosViewModel extends ChangeNotifier {
  late final getAgendamentosDoDiaCommand = Command1<List<Agendamento>, DateTime>(
    _getAgendamentosDoDia,
  );
  
  Future<ResultApp<List<Agendamento>>> _getAgendamentosDoDia(DateTime data) async {
    return await _getAgendamentosDoDiaUseCase.execute(data);
  }
}

// UI com ListenableBuilder
ListenableBuilder(
  listenable: viewModel.getAgendamentosDoDiaCommand,
  builder: (context, child) {
    final command = viewModel.getAgendamentosDoDiaCommand;
    if (command.running) return CircularProgressIndicator();
    if (command.error) return Text('Erro');
    return ListView(...);
  },
)
```

## 🎨 Melhorias Visuais

### Novo Card de Agendamento:
- ✅ Design mais limpo e organizado
- ✅ Melhor hierarquia visual
- ✅ Ícones mais intuitivos
- ✅ Tratamento de overflow de texto
- ✅ Sombras para profundidade

### Página de Exemplo:
- ✅ Card de resumo com estatísticas
- ✅ Indicador de loading no AppBar
- ✅ FloatingActionButton para refresh
- ✅ Melhor UX geral

## 📈 Próximos Passos

1. **Testar a refatoração** com dados reais
2. **Migrar outras funcionalidades** para o novo padrão
3. **Remover código legacy** após validação
4. **Atualizar documentação** geral do projeto
5. **Treinar equipe** no novo padrão

---

**Arquitetura**: App Architecture (Google/Flutter)  
**Padrão**: MVVM com ChangeNotifier + Commands  
**Data**: Janeiro 2025
