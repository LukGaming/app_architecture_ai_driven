# 📐 Regras de Arquitetura - App MP Prof

## 🎯 Objetivo

Este documento define as **regras obrigatórias** para manter a arquitetura limpa e consistente no projeto. Todas as implementações devem seguir essas diretrizes.

---

## 🏗️ Regras Fundamentais

### 1. **Nomenclatura Obrigatória**

#### ✅ **ViewModels (não Controllers)**
```dart
// ✅ CORRETO
class AgendamentosViewModel extends ChangeNotifier {}
class ClinicasViewModel extends ChangeNotifier {}

// ❌ PROIBIDO
class AgendamentosController extends StateNotifier {}
class ClinicasController extends ChangeNotifier {}
```

#### ✅ **Commands para Operações**
```dart
// ✅ CORRETO - Commands para operações específicas
class AgendamentosViewModel extends ChangeNotifier {
  late final Command0 carregarAgendamentos;
  late final Command1<DateTime, void> selecionarData;
  late final Command0 limparAgendamentos;
}

// ❌ PROIBIDO - Métodos síncronos para operações assíncronas
class AgendamentosViewModel {
  void carregarAgendamentos() async {} // ❌
  Future<void> carregarAgendamentos() async {} // ❌
}
```

#### ✅ **Otimização de Commands**

**Commands que executam somente uma linha devem referenciar diretamente o UseCase/Repository:**

```dart
// ✅ CORRETO - Referência direta quando há apenas uma chamada
class AuthViewModel extends ChangeNotifier {
  final AuthRepository _authRepository;
  final GetUserUsecase _getUserUsecase;

  late final loginUseCase = Command1(_loginUseCase.login);
  late final getUserUseCase = Command0(_getUserUsecase.get);      // Direto no UseCase
  late final logout = Command0(_authRepository.logout);         // Direto no Repository
}

// ❌ EVITAR - Métodos intermediários desnecessários
class AuthViewModel extends ChangeNotifier {
  late final getUserUseCase = Command0(_verifyLogingState);      // ❌ Método intermediário
  late final logout = Command0(_logout);                        // ❌ Método intermediário
  
  Future<ResultApp<User>> _verifyLogingState() async {          // ❌ Uma linha apenas
    return await _getUserUsecase.get();
  }
  
  Future<ResultApp<void>> _logout() async {                     // ❌ Uma linha apenas
    return await _authRepository.logout();
  }
}
```

---

### 2. **ReactiveBuilder vs StateBuilder**

#### ✅ **ReactiveBuilder para UI**
```dart
// ✅ CORRETO - ReactiveBuilder em páginas e widgets
ReactiveBuilder(
  builder: (context) {
    final viewModel = context.read<AgendamentosViewModel>();
    return ListView(...);
  },
)
```

#### ⚠️ **StateBuilder SOMENTE em Validadores**
```dart
// ✅ ÚNICO CASO PERMITIDO - Validação de formulários
StateBuilder<FormValidator>(
  builder: (context, validator) {
    return TextFormField(
      decoration: InputDecoration(
        errorText: validator.emailError,
      ),
    );
  },
)
```

---

### 3. **Isolamento de Repositórios**

#### ❌ **PROIBIDO - Repositório dependendo de outro repositório**
```dart
// ❌ NUNCA FAZER ISSO
class AgendamentosRepository {
  final ClinicasRepository _clinicasRepo; // ❌ VIOLAÇÃO
  final PacientesRepository _pacientesRepo; // ❌ VIOLAÇÃO
}
```

#### ✅ **CORRETO - Repositório isolado**
```dart
// ✅ PERMITIDO - Apenas serviços externos
class AgendamentosRepository {
  final MedplusApi _medplusApi; // ✅ Serviço externo
  final SharedPreferencesService _preferences; // ✅ Serviço externo
  
  // ❌ NUNCA outros repositórios
}
```

---

### 4. **API Service Centralizado**

#### ✅ **MedplusApi ÚNICO serviço de API**
```dart
// ✅ CORRETO - Um único serviço para todas as APIs
class MedplusApi {
  final Dio _dio; // Dio centralizado com interceptors
  
  Future<ResultApp<T>> login(...) async {}
  Future<ResultApp<T>> getAgendamentos(...) async {}
  Future<ResultApp<T>> getPacientes(...) async {}
  // Todos os métodos de API aqui
}
```

#### ❌ **PROIBIDO - Múltiplos serviços de API**
```dart
// ❌ NÃO CRIAR
class AgendamentosApiService {} // ❌
class PacientesApiService {} // ❌
class ClinicasApiService {} // ❌
```

---

### 5. **Injeção de Dependências via Router**

#### ✅ **Dependências Explícitas nas Rotas**
```dart
// ✅ CORRETO - router_config.dart
GoRoute(
  path: '/agendamentos',
  builder: (context, state) => AgendamentosPage(
    viewModel: AgendamentosViewModel(
      GetIt.instance<GetAgendamentosUseCase>(),
      GetIt.instance<AgendamentosRepository>(),
      // Dependências explícitas
    ),
  ),
)

// ✅ CORRETO - Widget recebendo ViewModel
class AgendamentosPage extends StatelessWidget {
  final AgendamentosViewModel viewModel;
  const AgendamentosPage({required this.viewModel});
}
```

#### ❌ **PROIBIDO - Service Locator nos Widgets**
```dart
// ❌ NÃO FAZER - Service Locator no widget
class AgendamentosPage extends StatelessWidget {
  Widget build(context) {
    final viewModel = GetIt.instance<AgendamentosViewModel>(); // ❌
    // Dependências implícitas
  }
}
```

---

### 6. **UseCases: Regras Fundamentais**

#### 📋 **Regras Obrigatórias dos UseCases:**

**1. ✅ Criar UseCases SOMENTE quando atender uma ou mais condições específicas**
**2. ❌ UseCases NÃO devem executar outros UseCases** 
**3. ❌ UseCases NÃO devem receber outros UseCases por injeção de dependências**

#### ✅ **Criar UseCases APENAS quando uma ou mais das condições existir:**

**🔀 Condição 1: Precisa combinar dados de múltiplos repositórios**
```dart
// ✅ UseCase NECESSÁRIO - combina dados de vários repositórios
class ProcessarAgendamentoCompletoUseCase {
  final AgendamentosRepository _agendamentosRepo;
  final PacientesRepository _pacientesRepo;
  final ClinicasRepository _clinicasRepo;
  final NotificacoesRepository _notificacoesRepo;
  
  Future<ResultApp<AgendamentoCompleto>> execute(int id) async {
    // Combina dados de 4 repositórios diferentes
    final agendamento = await _agendamentosRepo.getById(id);
    final paciente = await _pacientesRepo.getById(agendamento.pacienteId);
    final clinica = await _clinicasRepo.getById(agendamento.clinicaId);
    final notificacoes = await _notificacoesRepo.getByAgendamento(id);
    
    return AgendamentoCompleto.combinar(agendamento, paciente, clinica, notificacoes);
  }
}
```

**🧠 Condição 2: Lógica excessivamente complexa**
```dart
// ✅ UseCase NECESSÁRIO - lógica muito complexa
class CalcularEstatisticasAvancadasUseCase {
  final AgendamentosRepository _agendamentosRepo;
  
  Future<ResultApp<EstatisticasCompletas>> execute() async {
    final dados = await _agendamentosRepo.getTodosAgendamentos();
    
    // Algoritmos complexos de análise estatística
    final mediaTempoEspera = _calcularMediaComVariancia(dados);
    final desviosGeoLocalizacao = _analisarDistribuicaoGeografica(dados);
    final tendenciasSazonais = _analisarTendenciasTrimestre(dados);
    final predicoesMachineLearning = _gerarPredicoes(dados);
    
    return EstatisticasCompletas(
      mediaTempoEspera, 
      desviosGeoLocalizacao, 
      tendenciasSazonais, 
      predicoesMachineLearning,
    );
  }
}
```

**♻️ Condição 3: Lógica reutilizada por diferentes ViewModels**
```dart
// ✅ UseCase NECESSÁRIO - reutilização entre múltiplos ViewModels
class ValidarDisponibilidadeAgendamentoUseCase {
  final AgendamentosRepository _agendamentosRepo;
  final MedicosRepository _medicosRepo;
  
  Future<ResultApp<bool>> execute(DateTime data, int medicoId) async {
    // Lógica complexa reutilizada por:
    // - AgendamentosViewModel
    // - CalendarioViewModel 
    // - RelatoriosViewModel
    // - DisponibilidadeViewModel
    
    final conflitos = await _verificarConflitosAgenda(data, medicoId);
    final restricoesMedico = await _verificarRestricoesMedico(medicoId);
    final limitesInstitucionais = await _verificarLimitesInstitucionais(data);
    
    return _validarDisponibilidadeCompleta(conflitos, restricoesMedico, limitesInstitucionais);
  }
}
```

#### ❌ **NÃO criar UseCases quando:**

**1. Operação simples com apenas 1 repositório**
**2. Lógica simples de transformação** 
**3. Usado por apenas 1 ViewModel**
**4. Simples operações CRUD**

#### ❌ **ANTI-PATTERN - UseCase desnecessário:**
```dart
// ❌ DESNECESSÁRIO - só usa 1 repositório, lógica simples
class GetAgendamentosUseCase {
  final AgendamentosRepository _repository;
  
  Future<ResultApp<List<Agendamento>>> execute() async {
    return await _repository.getAgendamentos(); // Simples demais!
  }
}
```

#### ✅ **CORRETO - ViewModel fala diretamente com Repository:**
```dart
// ✅ PATTERN CORRETO - ViewModel → Repository direto
class AgendamentosViewModel extends ChangeNotifier {
  final AgendamentosRepository _agendamentosRepo;
  
  Future<void> _carregarAgendamentos() async {
    final result = await _agendamentosRepo.getAgendamentos();
    // Direto e simples - sem UseCase desnecessário
  }
}
```

#### ❌ **PROIBIDO - UseCase chamando outro UseCase:**
```dart
// ❌ ANTI-PATTERN - UseCase dependendo de outro UseCase
class SearchPacientesUseCase {
  final PacientesRepository _pacientesRepo;
  final GetClinicaSelecionadaUseCase _getClinicaUseCase; // ❌ PROIBIDO
  
  SearchPacientesUseCase(this._pacientesRepo, this._getClinicaUseCase);
  
  Future<ResultApp<List<Paciente>>> execute(String query) async {
    final clinica = _getClinicaUseCase.execute(); // ❌ UseCase executando UseCase
    return await _pacientesRepo.searchPacientes(query, clinica.id);
  }
}
```

#### ✅ **CORRETO - UseCase usando apenas Repositories:**
```dart
// ✅ PATTERN CORRETO - UseCase com responsabilidade única
class SearchPacientesUseCase {
  final PacientesRepository _pacientesRepo;
  final ClinicasRepository _clinicasRepo; // ✅ Repository direto
  
  SearchPacientesUseCase(this._pacientesRepo, this._clinicasRepo);
  
  Future<ResultApp<List<Paciente>>> execute(String query) async {
    final clinica = _clinicasRepo.getClinicaSelecionada(); // ✅ Direto do repository
    return await _pacientesRepo.searchPacientes(query, clinica.id);
  }
}
```

#### ✅ **UseCase NECESSÁRIO - lógica complexa:**
```dart
// ✅ UseCase NECESSÁRIO - lógica complexa entre múltiplos repositories
class ProcessarAgendamentoCompletoUseCase {
  final AgendamentosRepository _agendamentosRepo;
  final PacientesRepository _pacientesRepo;
  final ClinicasRepository _clinicasRepo;
  
  Future<ResultApp<AgendamentoCompleto>> execute(int id) async {
    // 1. Buscar agendamento
    final agendamento = await _agendamentosRepo.getById(id);
    
    // 2. Buscar dados do paciente
    final paciente = await _pacientesRepo.getById(agendamento.pacienteId);
    
    // 3. Buscar dados da clínica
    final clinica = await _clinicasRepo.getById(agendamento.clinicaId);
    
    // 4. Calcular tempo de espera
    final tempoEspera = _calcularTempoEspera(agendamento);
    
    // 5. Aplicar regras de negócio
    final status = _determinarStatus(agendamento, tempoEspera);
    
    // 6. Combinar tudo
    return AgendamentoCompleto(
      agendamento: agendamento,
      paciente: paciente,
      clinica: clinica,
      tempoEspera: tempoEspera,
      status: status,
    );
  }
}
```

#### 📝 **Justificativas das Regras dos UseCases:**

**Por que UseCases devem atender condições específicas?**

1. **🎯 Responsabilidade Clara**: UseCases só existem quando há valor real agregado
2. **🧪 Simplicidade**: ViewModels falam diretamente com Repositories quando possível
3. **🔧 Manutenibilidade**: Menos camadas = menos complexidade desnecessária
4. **⚡ Performance**: Evita camadas de abstração desnecessárias
5. **📐 Arquitetura Limpa**: Cada camada tem propósito claro e justificado

**Fluxos de Comunicação Corretos:**

**Para operações simples:**
```
ViewModel → Repository → API/Storage
```

**Para operações complexas (que atendem às 3 condições):**
```
ViewModel → UseCase → Repository(s) → API/Storage
```

**📊 Matriz de Decisão: Criar UseCase?**

| Condição | Descrição | Exemplo |
|----------|-----------|---------|
| 🔀 **Múltiplos Repos** | Combina dados de 2+ repositórios | AgendamentoCompleto (agendamento + paciente + clínica) |
| 🧠 **Lógica Complexa** | Algoritmos/cálculos complexos | Estatísticas avançadas, ML, análises |
| ♻️ **Reutilização** | Usado por 2+ ViewModels | Validação de disponibilidade |

**Resultado:**
- **✅ Alguma condição atendida** → Criar UseCase
- **❌ Nenhuma condição atendida** → ViewModel → Repository direto

#### ❌ **NÃO usar UseCases para operações simples**
```dart
// ❌ UseCase DESNECESSÁRIO - operação trivial
class GetAgendamentosUseCase {
  Future<ResultApp<List<Agendamento>>> execute() async {
    return await _repository.getAgendamentos(); // Trivial demais
  }
}

// ✅ FAZER DIRETO no ViewModel
class AgendamentosViewModel {
  void _carregarAgendamentos() async {
    final result = await _repository.getAgendamentos();
    // Simples e direto
  }
}
```

---

### 7. **Tratamento de Erros Centralizado**

#### ✅ **ResultApp em toda comunicação**
```dart
// ✅ CORRETO - APIs retornam ResultApp
Future<ResultApp<List<Agendamento>>> getAgendamentos() async {
  try {
    final response = await _dio.get('/agendamentos');
    return ResultApp.ok(agendamentos);
  } catch (e) {
    return ResultApp.error(Exception(e.toString()));
  }
}

// ✅ CORRETO - ViewModels tratam ResultApp
switch (result) {
  case Ok(:final value):
    _agendamentos = value;
  case Error(:final error):
    _showError(error);
}
```

---

### 8. **Refresh Token Automático**

#### ✅ **Sistema Transparente**
O refresh token é **automático** e **transparente**:

```dart
// ✅ Interceptor automático no Dio
onError: (error, handler) async {
  if (error.response?.statusCode == 401) {
    // 1. Detecta token expirado
    // 2. Faz refresh automaticamente
    // 3. Refaz requisição original
    // 4. Usuário não percebe
  }
}
```

**📋 Regra**: Nunca implementar refresh manual nos ViewModels.

---

## 🚫 Anti-Patterns (O que NÃO fazer)

### 1. **Controllers com múltiplas responsabilidades**
```dart
// ❌ ANTI-PATTERN
class MegaController {
  // Agendamentos
  void carregarAgendamentos() {}
  
  // Pacientes  
  void buscarPacientes() {}
  
  // Clínicas
  void selecionarClinica() {}
  
  // Autenticação
  void login() {}
}
```

### 2. **Repositórios acoplados**
```dart
// ❌ ANTI-PATTERN
class AgendamentosRepository {
  final ClinicasRepository _clinicas; // ❌
  final PacientesRepository _pacientes; // ❌
  
  Future<void> salvarAgendamento() async {
    final clinica = await _clinicas.get(); // ❌ Acoplamento
    final paciente = await _pacientes.get(); // ❌ Acoplamento
  }
}
```

### 3. **Service Locator em widgets**
```dart
// ❌ ANTI-PATTERN
class MyWidget extends StatelessWidget {
  Widget build(context) {
    final service1 = GetIt.instance<Service1>(); // ❌
    final service2 = GetIt.instance<Service2>(); // ❌
    final service3 = GetIt.instance<Service3>(); // ❌
  }
}
```

### 4. **UseCases triviais**
```dart
// ❌ ANTI-PATTERN - UseCase desnecessário
class SaveStringUseCase {
  Future<void> execute(String key, String value) async {
    await _storage.setString(key, value); // Trivial demais
  }
}
```

---

## ✅ Checklist de Conformidade

### Antes de fazer PR, verificar:

#### **ViewModels**
- [ ] Nome termina com `ViewModel` (não Controller)
- [ ] Usa Commands para operações assíncronas
- [ ] Recebe dependências via construtor (não GetIt interno)
- [ ] Herda de `ChangeNotifier`

#### **UI**
- [ ] Usa `ReactiveBuilder` (não StateBuilder, exceto validadores)
- [ ] Widgets recebem ViewModels via parâmetro
- [ ] Não usa `GetIt.instance` dentro dos widgets

#### **Repositórios**
- [ ] NÃO depende de outros repositórios
- [ ] Usa apenas `MedplusApi` para chamadas HTTP
- [ ] Retorna `ResultApp<T>` em todos os métodos

#### **API**
- [ ] Todos os métodos HTTP estão no `MedplusApi`
- [ ] NÃO criar novos serviços de API
- [ ] Refresh token funciona automaticamente

#### **UseCases**
- [ ] Criado SOMENTE para lógica complexa
- [ ] NÃO criado para operações triviais
- [ ] Nome descritivo da operação de negócio

---

## 🎯 Conclusão

Essas regras garantem:
- **Consistência** na base de código
- **Manutenibilidade** a longo prazo  
- **Testabilidade** de todos os componentes
- **Performance** otimizada
- **Escalabilidade** da arquitetura

**⚠️ IMPORTANTE**: Qualquer violação dessas regras deve ser justificada e aprovada pela equipe.
