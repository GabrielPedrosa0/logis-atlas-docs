# Logis Atlas, documentação técnica

Logis Atlas é um sistema de controle de almoxarifado com várias unidades, feito
para ambiente hospitalar. Está em produção no Hospital Municipal Raimundo Célio
Rodrigues, em Pacatuba (CE).

O código é privado porque o sistema roda em ambiente hospitalar com dados reais.
Este repositório explica o que o sistema faz e por que foi construído assim.

## O problema

Trabalhei no almoxarifado desse hospital antes de programar e vi onde o processo
travava. O Logis Atlas registra o saldo de cada material por unidade e por
setor, mostra o que está para vencer e calcula o que precisa ser comprado.

## Resultados

- 5.035 entradas e 2.340 saídas registradas em agosto de 2026.
- Inventário rotativo com relatório de divergência item a item.

## Stack

React 19, Vite, Tailwind CSS 4, Supabase (Auth, PostgreSQL com RLS, Storage e
Edge Functions), Recharts e deploy na Vercel. Não há backend Node separado. A
regra de negócio mora no banco, em funções e policies.

## O que o sistema faz

**Pedidos.** O solicitante de cada departamento entra com login próprio e pede
pelo catálogo, como num aplicativo de compras. O almoxarifado aprova item a
item: aprova tudo, corta a quantidade ou recusa com motivo. A entrega dá baixa
no estoque.

**Estoque.** Saldo por unidade e por setor. Entrada com fornecedor, nota, lote e
validade. Saída em três operações: consumo, transferência entre setores e
descarte com motivo. A transferência é atômica. Qualquer movimentação pode ser
estornada sem apagar o registro original.

**Controle.** Lote e validade com FEFO, para o primeiro a vencer ser o primeiro a
sair. Cota por departamento. Estoque mínimo, ponto de pedido e alerta de pedido
acima da média histórica.

**Análise.** Consumo médio diário, cobertura em dias e sugestão de compra
arredondada para embalagem fechada. Alerta de vencimento. Relatórios em PDF,
XLSX e CSV.

## Modelo de dados

```
estabelecimento   unidade hospitalar, o tenant
└── setor          ponto de estoque físico, com saldo próprio
    └── saldo      quantidade por produto e setor
```

Produtos seguem uma classificação em árvore com código hierárquico:

```
03           EPI e segurança          grupo
03.01        Proteção das mãos        subgrupo
03.01.0007   Luva nitrílica           produto
```

Antes, a categoria era texto livre e apareciam "EPI", "EPIs" e "E.P.I." como
categorias diferentes. Nenhum relatório por categoria fechava. A árvore resolve
isso, e a numeração de cada produto pertence ao subgrupo, então o código é único
por construção.

## Perfis

| Perfil | O que pode |
|---|---|
| Desenvolvedor | Vê todas as unidades e nunca movimenta estoque |
| Administrador | Cadastros, usuários e plano da unidade |
| Setor | Opera o estoque |
| Solicitante | Faz pedidos e não vê saldo |

## Decisões

**A RLS é a única barreira.** O cadastro é aberto, então uma unidade pode ser
hostil de propósito, e a chave pública do Supabase está no navegador. Toda regra
de acesso está no banco.

**Toda função privilegiada checa permissão na primeira linha.** As regras ficam
centralizadas em poucas funções auxiliares, reaproveitadas pelas policies.

**Toda view respeita a RLS de quem consulta.** Sem isso, a view roda com os
privilégios do dono e ignora as permissões das tabelas de origem.

**O perfil do usuário só muda por função no servidor.** Se a tabela de usuários
aceitasse update direto, uma pessoa poderia se promover a administrador em uma
requisição.

**Limite de plano é trigger, não tela.** O trigger vale até para a Edge Function
que roda com privilégio de serviço.

**O QR da etiqueta não usa o código do produto.** O código é sequencial e daria
para adivinhar os outros. O QR aponta para um token aleatório de 128 bits, que
pode ser trocado se uma etiqueta vazar.

## Etiquetas

O sistema gera etiquetas de prateleira em folha adesiva A4, com o código
hierárquico, o endereço de armazenagem e o QR. A migração dos códigos antigos e
a troca das etiquetas no almoxarifado estão em andamento.

## Segurança

Uma auditoria registrou onze achados, com três falsos positivos descartados.
Entre os corrigidos: biblioteca de PDF com vulnerabilidades conhecidas,
ausência de cabeçalhos de segurança HTTP, senha mínima curta, CORS aberto nas
Edge Functions e um campo de CPF guardado sem finalidade, que foi removido.

## Próximos passos

- Testes automatizados das operações de estoque no banco.
- Paginação nas listas grandes.
- Termos de uso e política de privacidade, exigidos pela LGPD com o cadastro
  aberto.

## Autor

Gabriel Pedrosa. [Portfólio](https://gabrielpedrosa0.github.io) ·
[LinkedIn](https://www.linkedin.com/in/gabriel-pedrosa-6618a6343/)
