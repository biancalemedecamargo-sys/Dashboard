import React, { createContext, useState, useMemo, useEffect, useContext, ReactNode } from 'react';

// --- Tipagens (sem alterações) ---
interface Transaction { id: number; description: string; amount: number; type: string; frequencia?: string; category: string; valorMedioMensal: number; date: string; }
interface Investment { id: number; name: string; amount: number; type: string; returnRate: number; period: number; date: string; }
interface Debt { id: number; name: string; amount: number; interestRate: number; minPayment: number; }
interface Params { taxaRetorno: number; taxaInflacao: number; caixa: number; nomeUsuario: string; mesesReserva: number; metaPatrimonio: number; }
interface Summary { receitasMensais: number; despesasMensais: number; fluxoMensal: number; investimentosTotal: number; dividasTotal: number; patrimonioTotal: number; patrimonioLiquido: number; taxaPoupanca: number; byCategoryExpense: Record<string, number>; byCategoryIncome: Record<string, number>; projecao: { mes: number, valor: number }[]; patrimonioFuturo: number; mesesParaEmergencia: number; mesesParaMeta: number; reservaAtual: number; metaEmergencia: number; progressoReserva: number; progressoMetaPatrimonio: number; }
interface FormData { description: string; amount: string; type: string; frequencia: string; category: string; }
interface InvestmentFormData { name: string; amount: string; type: string; returnRate: string; period: string; }
interface DebtFormData { name: string; amount: string; interestRate: string; minPayment: string; }
interface Goal { id: number; name: string; targetAmount: number; currentAmount: number; deadline?: string; category?: string; isCompleted: boolean; createdAt: string; }
interface GoalFormData { name: string; targetAmount: string; deadline?: string; category?: string; }

// --- Constantes Padrão ---
const DEFAULT_PARAMS: Params = { taxaRetorno: 10, taxaInflacao: 5, caixa: 10000, nomeUsuario: 'Usuário', mesesReserva: 6, metaPatrimonio: 1000000 };
const DEFAULT_FORM_DATA: FormData = { description: '', amount: '', type: 'income-rec', frequencia: '1', category: 'Salário' };
const DEFAULT_INVESTMENT_FORM: InvestmentFormData = { name: '', amount: '', type: 'renda-fixa', returnRate: '10', period: '12' };
const DEFAULT_DEBT_FORM: DebtFormData = { name: '', amount: '', interestRate: '5', minPayment: '0' };

// --- Função de Migração de Parâmetros ---
const migrateParams = (p: any): Params => {
    // Garante que 'p' seja um objeto antes de tentar acessar propriedades
    const safeP = (p && typeof p === 'object') ? p : {};
    const caixa = safeP.caixa ?? safeP.patrimonioInicial ?? DEFAULT_PARAMS.caixa;
    return {
      caixa: parseFloat(String(caixa)) || DEFAULT_PARAMS.caixa, // Fallback robusto
      nomeUsuario: String(safeP.nomeUsuario || DEFAULT_PARAMS.nomeUsuario),
      mesesReserva: parseInt(String(safeP.mesesReserva)) || DEFAULT_PARAMS.mesesReserva,
      metaPatrimonio: parseFloat(String(safeP.metaPatrimonio)) || DEFAULT_PARAMS.metaPatrimonio,
      taxaRetorno: parseFloat(String(safeP.taxaRetorno)) || DEFAULT_PARAMS.taxaRetorno,
      taxaInflacao: parseFloat(String(safeP.taxaInflacao)) || DEFAULT_PARAMS.taxaInflacao,
    };
};

interface DashboardContextProps { /* ... (igual antes) ... */ transactions: Transaction[]; investments: Investment[]; debts: Debt[]; goals: Goal[]; params: Params; summary: Summary | null; formData: FormData; investmentForm: InvestmentFormData; debtForm: DebtFormData; setTransactions: React.Dispatch<React.SetStateAction<Transaction[]>>; setInvestments: React.Dispatch<React.SetStateAction<Investment[]>>; setDebts: React.Dispatch<React.SetStateAction<Debt[]>>; setGoals: React.Dispatch<React.SetStateAction<Goal[]>>; setParams: React.Dispatch<React.SetStateAction<Params>>; setFormData: React.Dispatch<React.SetStateAction<FormData>>; setInvestmentForm: React.Dispatch<React.SetStateAction<InvestmentFormData>>; setDebtForm: React.Dispatch<React.SetStateAction<DebtFormData>>; addTransaction: () => void; deleteTransaction: (id: number) => void; addInvestment: () => void; deleteInvestment: (id: number) => void; addDebt: () => void; deleteDebt: (id: number) => void; addGoal: (goalData: Omit<Goal, 'id' | 'currentAmount' | 'isCompleted' | 'createdAt'>) => void; deleteGoal: (id: number) => void; calculateNPER: (rate: number, pmt: number, pv: number, fv: number) => number; }
const DashboardContext = createContext<DashboardContextProps | undefined>(undefined);
export const useDashboard = () => { const context = useContext(DashboardContext); if (!context) { throw new Error('useDashboard must be used within a DashboardProvider'); } return context; };

interface DashboardProviderProps { children: ReactNode; }

export const DashboardProvider: React.FC<DashboardProviderProps> = ({ children }) => {
  // Estados usando Constantes Padrão
  const [transactions, setTransactions] = useState<Transaction[]>([]);
  const [investments, setInvestments] = useState<Investment[]>([]);
  const [debts, setDebts] = useState<Debt[]>([]);
  const [goals, setGoals] = useState<Goal[]>([]);
  const [params, setParams] = useState<Params>(DEFAULT_PARAMS);
  const [formData, setFormData] = useState<FormData>(DEFAULT_FORM_DATA);
  const [investmentForm, setInvestmentForm] = useState<InvestmentFormData>(DEFAULT_INVESTMENT_FORM);
  const [debtForm, setDebtForm] = useState<DebtFormData>(DEFAULT_DEBT_FORM);

  // Load from localStorage com Validação Robusta
  useEffect(() => {
    const saved = localStorage.getItem('w1_data');
    if (!saved) return; // Sai cedo se não houver dados salvos

    try {
      const data = JSON.parse(saved);
      // Valida a estrutura geral
      if (data && typeof data === 'object') {
        setTransactions(Array.isArray(data.transactions) ? data.transactions : []);
        setInvestments(Array.isArray(data.investments) ? data.investments : []);
        setDebts(Array.isArray(data.debts) ? data.debts : []);
        setGoals(Array.isArray(data.goals) ? data.goals : []);
        // Usa a função de migração para params
        setParams(migrateParams(data.params));
      } else {
        // Se a estrutura geral não for um objeto, limpa
        throw new Error("Dados salvos em formato inválido.");
      }
    } catch (error) {
      console.error("Erro ao carregar ou validar dados do localStorage:", error);
      // Limpa dados corrompidos para evitar erros futuros
      localStorage.removeItem('w1_data');
      // Reseta para os padrões (opcional, mas seguro)
      setTransactions([]); setInvestments([]); setDebts([]); setGoals([]); setParams(DEFAULT_PARAMS);
    }
  }, []); // Executa apenas na montagem inicial

  // Save to localStorage (sem alterações na lógica interna)
  useEffect(() => { try { localStorage.setItem('w1_data', JSON.stringify({ transactions, investments, params, debts, goals })); } catch (error) { console.error("Erro ao salvar dados no localStorage:", error); } }, [transactions, investments, params, debts, goals]);

  // Função NPER (sem alterações)
  const calculateNPER = (rate: number, pmt: number, pv: number, fv: number): number => { const r = rate; const P = pmt; const V = pv; const F = fv; if (r === 0 || r <= -1) { if (P <= 0 && V < F) return Infinity; if (P <= 0) return Infinity; if (P === 0 && V >= F) return 0; if (P === 0) return Infinity; return (F - V) / P; } if ((V * r + P <= 0) && V < F) { return Infinity; } const numerador = F * r + P; const denominador = V * r + P; if (denominador <= 0) { if (numerador > 0 && r > 0) return Infinity; if (V >= F) return 0; return Infinity; } const ratio = numerador / denominador; if (ratio <= 0) { if (V >= F) return 0; return Infinity; } if (1 + r <= 0) return Infinity; const n = Math.log(ratio) / Math.log(1 + r); return n >= 0 ? n : 0; };

  // Cálculos Memoizados (removido calculateNPER das dependências)
  const summary = useMemo((): Summary | null => {
    // ... (lógica interna do useMemo exatamente igual à versão anterior) ...
    if (!params) return null; const receitas = transactions.filter(t => t.type?.includes('income')).reduce((sum, t) => sum + (parseFloat(String(t.valorMedioMensal)) || 0), 0); const despesas = transactions.filter(t => t.type?.includes('expense')).reduce((sum, t) => sum + (parseFloat(String(t.valorMedioMensal)) || 0), 0); const investimentosTotal = investments.reduce((sum, inv) => sum + parseFloat(String(inv.amount || 0)), 0); const dividasTotal = debts.reduce((sum, d) => sum + parseFloat(String(d.amount || 0)), 0); const caixaAtual = parseFloat(String(params.caixa)) || 0; const patrimonioTotal = investimentosTotal + caixaAtual; const patrimonioLiquido = patrimonioTotal - dividasTotal; const byCategoryExpense: Record<string, number> = {}; transactions.filter(t => t.type === 'expense-rec').forEach(t => { const category = t.category || 'Sem Categoria'; if (!byCategoryExpense[category]) byCategoryExpense[category] = 0; byCategoryExpense[category] += parseFloat(String(t.valorMedioMensal || 0)); }); const byCategoryIncome: Record<string, number> = {}; transactions.filter(t => t.type === 'income-rec').forEach(t => { const category = t.category || 'Sem Categoria'; if (!byCategoryIncome[category]) byCategoryIncome[category] = 0; byCategoryIncome[category] += parseFloat(String(t.valorMedioMensal || 0)); }); const fluxoMensal = receitas - despesas; const taxaRetornoAnual = parseFloat(String(params.taxaRetorno)) || 0; const taxaMensal = (taxaRetornoAnual > -100) ? (Math.pow(1 + taxaRetornoAnual / 100, 1/12) - 1) : 0; let patrimonio = patrimonioTotal; const projecao: { mes: number, valor: number }[] = []; const fatorCrescimento = 1 + taxaMensal; for (let i = 0; i <= 12; i++) { projecao.push({ mes: i, valor: patrimonio }); patrimonio = (patrimonio * fatorCrescimento) + fluxoMensal; } const mesesReservaParam = parseInt(String(params.mesesReserva)) || 0; const metaEmergencia = despesas * mesesReservaParam; const reservaAtual = caixaAtual + investments.filter(inv => inv.type === 'renda-fixa').reduce((sum, inv) => sum + parseFloat(String(inv.amount || 0)), 0); const mesesParaEmergencia = metaEmergencia > reservaAtual ? calculateNPER(taxaMensal, fluxoMensal, reservaAtual, metaEmergencia) : 0; const metaPatrimonioParam = parseFloat(String(params.metaPatrimonio)) || 0; const mesesParaMeta = metaPatrimonioParam > patrimonioTotal ? calculateNPER(taxaMensal, fluxoMensal, patrimonioTotal, metaPatrimonioParam) : 0; const progressoReserva = metaEmergencia > 0 ? Math.min((reservaAtual / metaEmergencia) * 100, 100) : (metaEmergencia <= 0 ? 100 : 0); const progressoMetaPatrimonio = metaPatrimonioParam > 0 ? Math.min((patrimonioTotal / metaPatrimonioParam) * 100, 100) : (metaPatrimonioParam <= 0 ? 100: 0) ; return { receitasMensais: receitas, despesasMensais: despesas, fluxoMensal, investimentosTotal, dividasTotal, patrimonioTotal, patrimonioLiquido, taxaPoupanca: receitas > 0 ? (fluxoMensal / receitas * 100) : 0, byCategoryExpense, byCategoryIncome, projecao, patrimonioFuturo: projecao[12]?.valor || patrimonioTotal, mesesParaEmergencia, mesesParaMeta, reservaAtual, metaEmergencia, progressoReserva, progressoMetaPatrimonio };
  }, [transactions, investments, params, debts]); // calculateNPER removido

  // --- Funções add/delete com try/catch e confirmação ---

  const addTransaction = () => {
    try {
      if (!formData.description?.trim()) throw new Error('Descrição é obrigatória.');
      const amount = parseFloat(formData.amount);
      if (isNaN(amount) || amount <= 0) throw new Error('Valor deve ser um número positivo.');
      const frequencia = parseInt(formData.frequencia || '1');
      if (isNaN(frequencia) || frequencia <= 0) throw new Error('Frequência inválida.');

      let valorMedioMensal = 0;
      if (formData.type.includes('rec')) {
        valorMedioMensal = amount / frequencia;
      }
      const newTransaction: Transaction = { id: Date.now(), ...formData, amount, valorMedioMensal, date: new Date().toLocaleDateString('pt-BR') };

      if (formData.type.includes('unica')) {
        const newCaixa = formData.type === 'income-unica' ? params.caixa + amount : params.caixa - amount;
        if (newCaixa < 0 && formData.type === 'expense-unica') {
             // Opcional: Impedir caixa negativo em despesa única?
             // throw new Error("Caixa ficaria negativo com esta despesa única.");
        }
        setParams(prevParams => ({...prevParams, caixa: newCaixa}));
      }
      setTransactions(prev => [...prev, newTransaction]);
      setFormData(DEFAULT_FORM_DATA); // Reseta usando constante
    } catch (error: any) {
      alert(`Erro ao adicionar transação: ${error.message}`);
      console.error('Erro em addTransaction:', error);
    }
  };

  const deleteTransaction = (id: number) => {
    const transactionToDelete = transactions.find(t => t.id === id);
    if (!transactionToDelete) return;
    // Adiciona confirmação
    if (!window.confirm(`Tem certeza que deseja excluir a transação "${transactionToDelete.description}"?`)) return;
    try {
      if (transactionToDelete.type.includes('unica')) {
        const amount = parseFloat(String(transactionToDelete.amount || 0));
        const newCaixa = transactionToDelete.type === 'income-unica' ? params.caixa - amount : params.caixa + amount;
        setParams(prevParams => ({...prevParams, caixa: newCaixa}));
      }
      setTransactions(prev => prev.filter(t => t.id !== id));
    } catch (error: any) {
       alert(`Erro ao excluir transação: ${error.message}`);
       console.error('Erro em deleteTransaction:', error);
    }
  };

  const addInvestment = () => {
    try {
      const amount = parseFloat(investmentForm.amount);
      if (!investmentForm.name?.trim()) throw new Error('Nome do investimento é obrigatório.');
      if (isNaN(amount) || amount <= 0) throw new Error('Valor do investimento deve ser positivo.');
      if (amount > params.caixa) throw new Error(`Caixa insuficiente. Disponível: ${formatCurrency(params.caixa)}`);

      const newInvestment: Investment = { id: Date.now(), ...investmentForm, amount: amount, returnRate: parseFloat(investmentForm.returnRate || '0'), period: parseInt(investmentForm.period || '0'), date: new Date().toLocaleDateString('pt-BR') };
      setInvestments(prev => [...prev, newInvestment]);
      setParams(prev => ({...prev, caixa: prev.caixa - amount}));
      setInvestmentForm(DEFAULT_INVESTMENT_FORM); // Reseta usando constante
    } catch (error: any) {
      alert(`Erro ao adicionar investimento: ${error.message}`);
      console.error('Erro em addInvestment:', error);
    }
  };

  const deleteInvestment = (id: number) => {
    const invToDelete = investments.find(inv => inv.id === id);
    if (!invToDelete) return;
    if (!window.confirm(`Tem certeza que deseja excluir o investimento "${invToDelete.name}"? O valor retornará ao caixa.`)) return;
    try {
        const amount = parseFloat(String(invToDelete.amount || 0));
        setInvestments(prev => prev.filter(inv => inv.id !== id));
        setParams(prev => ({...prev, caixa: prev.caixa + amount}));
    } catch (error: any) {
       alert(`Erro ao excluir investimento: ${error.message}`);
       console.error('Erro em deleteInvestment:', error);
    }
  };

  const addDebt = () => {
    try {
      const amount = parseFloat(debtForm.amount);
      if (!debtForm.name?.trim()) throw new Error('Nome da dívida é obrigatório.');
      if (isNaN(amount) || amount <= 0) throw new Error('Saldo devedor deve ser positivo.');
      const interestRate = parseFloat(debtForm.interestRate || '0');
      const minPayment = parseFloat(debtForm.minPayment || '0');
      if (isNaN(interestRate) || interestRate < 0) throw new Error('Taxa de juros inválida.');
      if (isNaN(minPayment) || minPayment < 0) throw new Error('Pagamento mínimo inválido.');

      const newDebt: Debt = { id: Date.now(), ...debtForm, amount: amount, interestRate: interestRate, minPayment: minPayment };
      setDebts(prev => [...prev, newDebt]);
      setDebtForm(DEFAULT_DEBT_FORM); // Reseta usando constante
    } catch (error: any) {
      alert(`Erro ao adicionar dívida: ${error.message}`);
      console.error('Erro em addDebt:', error);
    }
  };

  const deleteDebt = (id: number) => {
    const debtToDelete = debts.find(d => d.id === id);
    if (!debtToDelete) return;
    if (!window.confirm(`Tem certeza que deseja excluir a dívida "${debtToDelete.name}"?`)) return;
    try {
       setDebts(prev => prev.filter(d => d.id !== id));
    } catch (error: any) {
       alert(`Erro ao excluir dívida: ${error.message}`);
       console.error('Erro em deleteDebt:', error);
    }
  };

  const addGoal = (goalData: Omit<Goal, 'id' | 'currentAmount' | 'isCompleted' | 'createdAt'>) => {
    try {
        if (!goalData.name?.trim()) throw new Error('Nome da meta é obrigatório.');
        const targetAmount = parseFloat(String(goalData.targetAmount)) || 0;
        if (isNaN(targetAmount) || targetAmount <= 0) throw new Error("Valor da meta deve ser positivo.");

        const newGoal: Goal = {
            id: Date.now(),
            name: goalData.name,
            targetAmount: targetAmount,
            currentAmount: 0,
            deadline: goalData.deadline || undefined,
            category: goalData.category || undefined,
            isCompleted: false,
            createdAt: new Date().toLocaleDateString('pt-BR'),
        };
        setGoals(prev => [...prev, newGoal]);
    } catch (error: any) {
        alert(`Erro ao adicionar meta: ${error.message}`);
        console.error('Erro em addGoal:', error);
    }
  };

  const deleteGoal = (id: number) => {
    const goalToDelete = goals.find(g => g.id === id);
    if (!goalToDelete) return;
    if (!window.confirm(`Tem certeza que deseja excluir a meta "${goalToDelete.name}"?`)) return;
    try {
      setGoals(prev => prev.filter(g => g.id !== id));
    } catch (error: any) {
       alert(`Erro ao excluir meta: ${error.message}`);
       console.error('Erro em deleteGoal:', error);
    }
  };


  // Valor fornecido ao Contexto
  const value = {
    transactions, investments, debts, goals, params, summary, formData, investmentForm, debtForm,
    setTransactions, setInvestments, setDebts, setGoals, setParams, setFormData, setInvestmentForm, setDebtForm,
    addTransaction, deleteTransaction, addInvestment, deleteInvestment, addDebt, deleteDebt, addGoal, deleteGoal,
    calculateNPER
  };

  return <DashboardContext.Provider value={value}>{children}</DashboardContext.Provider>;
};


