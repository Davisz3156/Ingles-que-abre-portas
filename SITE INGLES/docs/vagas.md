# Guia de Configuração e Atualização de Vagas

Este documento orienta como gerenciar a exibição do bloco de vagas e o indicador de preenchimento da turma na página de oferta do **Inglês que Abre Portas**.

---

## 1. Onde Fica a Configuração

No arquivo `index.html`, localize o bloco de configuração `const CFG = { ... }` (por volta da linha 420):

```javascript
const CFG = {
  CHECKOUT_URL: "#",
  OFFER_VALUE: 39.99,
  PRECO_DE: "R$ 274,99",
  PRECO_POR: "R$ 39,99",
  VAGAS: 272,                   /* limite total de vagas da turma (0 esconde o bloco) */
  VAGAS_PREENCHIDAS: null,      /* null = sem barra de progresso; número = exibe barra real */
  PESSOAS_TESTE_SEMANA: null,   /* null = não exibe contagem; número = exibe valor fixo real */
  ...
};
```

---

## 2. Como Funciona a Exibição

### A) Modo Padrão (Sem Barra de Progresso)
Quando você define:
```javascript
VAGAS_PREENCHIDAS: null,
```
- O cartão azul-marinho sofisticado é exibido com:
  - Título: **"Turma limitada a 272 vagas"**
  - Subtítulo: **"Quando as vagas acabarem, o acesso fecha e o preço volta ao normal."**
- **Nenhuma barra de progresso nem porcentagens fictícias** são mostradas. O design permanece limpo, honesto e elegante.

---

### B) Modo com Vagas Preenchidas (Dado Real)
Quando você tiver o número exato de alunos matriculados e desejar exibir a barra de progresso:
```javascript
VAGAS_PREENCHIDAS: 80,
```
- A barra de progresso sutil (gradiente azul `#38BDF8` a `#0060F8`) será exibida automaticamente com preenchimento proporcional (80 de 272 = 29%).
- É exibido o texto: **"80 de 272 vagas preenchidas"**.
- É incluída acessibilidade para leitores de tela (`role="progressbar"`, `aria-valuenow="80"`, `aria-valuemin="0"`, `aria-valuemax="272"`).
- O número **nunca muda sozinho**, não diminui com o tempo e nunca é gerado aleatoriamente: reflete estritamente o valor que você digitar em `CFG.VAGAS_PREENCHIDAS`.

---

### C) Como Esconder o Bloco Inteiro
Se você não quiser exibir nenhuma menção a limite de vagas:
```javascript
VAGAS: 0,
```
- O bloco `.urgency-bar` é totalmente omitido do DOM.

---

## 3. Boas Práticas e Conformidade Legal
- Nunca utilize scripts de contagem regressiva artificial ou decremento falso de vagas.
- Mantenha `VAGAS_PREENCHIDAS` alinhado ao relatório de vendas do seu checkout para manter total transparência com os alunos.

