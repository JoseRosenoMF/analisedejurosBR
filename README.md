#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
JUROSBR - Analisador de Empréstimos
===================================

Calcula:
    • Juros simples
    • Juros compostos
    • Parcela de empréstimo (sistema Price)
    • Total pago e total de juros
    • Custo efetivo aproximado quando existem taxas adicionais
    • Comparação com uma taxa de referência do Banco Central
    • Diagnóstico de custo do empréstimo

IMPORTANTE:
O Brasil não possui uma única "taxa máxima de juros" válida para todos
os empréstimos. As taxas variam por modalidade, instituição, cliente,
prazo, garantias e outros fatores.

Este programa usa uma TAXA DE REFERÊNCIA informada pelo usuário.
Também oferece como exemplo uma referência de 63,0% a.a., correspondente
à taxa média do crédito livre para pessoas físicas divulgada pelo Banco
Central para abril de 2026. Essa referência não é teto legal.

Autor: José Roseno de Mendonça Filho
Licença: MIT
"""

from __future__ import annotations

import math
from dataclasses import dataclass
from enum import Enum


# ============================================================
# MODELOS
# ============================================================

class Sistema(str, Enum):
    SIMPLES = "simples"
    COMPOSTO = "composto"


@dataclass
class Emprestimo:
    principal: float
    taxa_anual: float
    meses: int
    taxas_extras: float = 0.0


@dataclass
class Resultado:
    parcela: float
    total_pago: float
    juros: float
    custo_total: float
    taxa_mensal_equivalente: float
    taxa_anual_efetiva: float
    percentual_sobre_principal: float


# ============================================================
# CONVERSÃO DE TAXAS
# ============================================================

def taxa_mensal_para_anual_efetiva(taxa_mensal: float) -> float:
    """
    Converte taxa mensal em taxa anual efetiva.

    Exemplo:
        2% a.m. -> (1.02 ** 12) - 1 = 26,82% a.a.
    """
    return (1 + taxa_mensal) ** 12 - 1


def taxa_anual_para_mensal_efetiva(taxa_anual: float) -> float:
    """
    Converte taxa anual efetiva em taxa mensal equivalente.

    Não divide simplesmente por 12.
    """
    return (1 + taxa_anual) ** (1 / 12) - 1


# ============================================================
# JUROS SIMPLES
# ============================================================

def juros_simples(principal: float, taxa: float, periodos: int) -> tuple[float, float]:
    """
    Retorna (juros, montante).

    M = P * (1 + i*n)
    J = M - P
    """
    if principal <= 0:
        raise ValueError("O principal deve ser maior que zero.")
    if taxa < 0:
        raise ValueError("A taxa não pode ser negativa.")
    if periodos <= 0:
        raise ValueError("O número de períodos deve ser maior que zero.")

    juros = principal * taxa * periodos
    montante = principal + juros

    return juros, montante


# ============================================================
# JUROS COMPOSTOS
# ============================================================

def juros_compostos(
    principal: float,
    taxa: float,
    periodos: int,
) -> tuple[float, float]:
    """
    Retorna (juros, montante).

    M = P * (1+i)^n
    J = M-P
    """
    if principal <= 0:
        raise ValueError("O principal deve ser maior que zero.")
    if taxa < 0:
        raise ValueError("A taxa não pode ser negativa.")
    if periodos <= 0:
        raise ValueError("O número de períodos deve ser maior que zero.")

    montante = principal * (1 + taxa) ** periodos
    juros = montante - principal

    return juros, montante


# ============================================================
# SISTEMA PRICE
# ============================================================

def parcela_price(principal: float, taxa_mensal: float, meses: int) -> float:
    """
    Calcula a parcela fixa do sistema Price.

    PMT = P * i * (1+i)^n / ((1+i)^n - 1)
    """
    if principal <= 0:
        raise ValueError("O principal deve ser maior que zero.")
    if meses <= 0:
        raise ValueError("O prazo deve ser maior que zero.")
    if taxa_mensal < 0:
        raise ValueError("A taxa mensal não pode ser negativa.")

    if taxa_mensal == 0:
        return principal / meses

    fator = (1 + taxa_mensal) ** meses
    return principal * taxa_mensal * fator / (fator - 1)


def analisar_emprestimo(emprestimo: Emprestimo) -> Resultado:
    """
    Analisa um empréstimo com parcelas fixas (Price).
    """
    taxa_mensal = taxa_anual_para_mensal_efetiva(
        emprestimo.taxa_anual
    )

    parcela = parcela_price(
        emprestimo.principal,
        taxa_mensal,
        emprestimo.meses,
    )

    total_pago = parcela * emprestimo.meses
    juros = total_pago - emprestimo.principal
    custo_total = total_pago + emprestimo.taxas_extras

    percentual = (
        (custo_total / emprestimo.principal) - 1
    ) * 100

    return Resultado(
        parcela=parcela,
        total_pago=total_pago,
        juros=juros,
        custo_total=custo_total,
        taxa_mensal_equivalente=taxa_mensal,
        taxa_anual_efetiva=emprestimo.taxa_anual,
        percentual_sobre_principal=percentual,
    )


# ============================================================
# ANÁLISE DE CUSTO
# ============================================================

def classificar_taxa(
    taxa_proposta_anual: float,
    taxa_referencia_anual: float,
) -> str:
    """
    Compara a proposta com uma referência.

    NÃO determina legalidade.
    Apenas classifica matematicamente a proposta em relação
    ao parâmetro escolhido.
    """
    if taxa_proposta_anual <= taxa_referencia_anual * 0.80:
        return "MUITO ABAIXO DA REFERÊNCIA"

    if taxa_proposta_anual <= taxa_referencia_anual:
        return "ABAIXO OU IGUAL À REFERÊNCIA"

    if taxa_proposta_anual <= taxa_referencia_anual * 1.25:
        return "ACIMA DA REFERÊNCIA"

    return "MUITO ACIMA DA REFERÊNCIA"


def analisar_decisao(
    emprestimo: Emprestimo,
    taxa_referencia_anual: float,
) -> dict:
    """
    Produz um diagnóstico informativo.

    O programa não diz simplesmente "pegue" ou "não pegue".
    Ele mostra os números que permitem avaliar a proposta.
    """
    resultado = analisar_emprestimo(emprestimo)

    diferenca = (
        emprestimo.taxa_anual - taxa_referencia_anual
    )

    diferenca_percentual = (
        diferenca / taxa_referencia_anual
    ) * 100

    return {
        "classificacao": classificar_taxa(
            emprestimo.taxa_anual,
            taxa_referencia_anual,
        ),
        "taxa_proposta": emprestimo.taxa_anual,
        "taxa_referencia": taxa_referencia_anual,
        "diferenca_pontos_percentuais": diferenca * 100,
        "diferenca_percentual": diferenca_percentual,
        "resultado": resultado,
    }


# ============================================================
# COMPARADOR DE DUAS PROPOSTAS
# ============================================================

def comparar_propostas(
    proposta_a: Emprestimo,
    proposta_b: Emprestimo,
) -> dict:
    """
    Compara duas propostas pelo custo total.
    """
    a = analisar_emprestimo(proposta_a)
    b = analisar_emprestimo(proposta_b)

    return {
        "proposta_a": a,
        "proposta_b": b,
        "economia_da_menor": abs(a.custo_total - b.custo_total),
    }


# ============================================================
# RELATÓRIO
# ============================================================

def dinheiro(valor: float) -> str:
    return f"R$ {valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")


def percentual(valor: float) -> str:
    return f"{valor * 100:.2f}%".replace(".", ",")


def imprimir_relatorio(
    emprestimo: Emprestimo,
    referencia: float,
) -> None:

    analise = analisar_decisao(
        emprestimo,
        referencia,
    )

    resultado: Resultado = analise["resultado"]

    print("\n" + "=" * 68)
    print("                 JUROSBR — ANÁLISE DO EMPRÉSTIMO")
    print("=" * 68)

    print(f"Valor solicitado........: {dinheiro(emprestimo.principal)}")
    print(
        f"Taxa contratada.........: {percentual(emprestimo.taxa_anual)} a.a.")
    print(
        f"Taxa mensal equivalente: {percentual(resultado.taxa_mensal_equivalente)} a.m.")
    print(f"Prazo...................: {emprestimo.meses} meses")
    print(f"Taxas extras............: {dinheiro(emprestimo.taxas_extras)}")

    print("\n--- SISTEMA PRICE ---")
    print(f"Parcela aproximada......: {dinheiro(resultado.parcela)}")
    print(f"Total das parcelas.....: {dinheiro(resultado.total_pago)}")
    print(f"Juros pagos............: {dinheiro(resultado.juros)}")
    print(f"Custo total.............: {dinheiro(resultado.custo_total)}")
    print(
        f"Custo acima do principal: "
        f"{resultado.percentual_sobre_principal:.2f}%"
    )

    print("\n--- COMPARAÇÃO COM REFERÊNCIA ---")
    print(f"Referência..............: {percentual(referencia)} a.a.")
    print(
        f"Diferença...............: "
        f"{analise['diferenca_pontos_percentuais']:.2f} pontos percentuais"
    )
    print(
        f"Variação relativa......: "
        f"{analise['diferenca_percentual']:.2f}%"
    )

    print("\nDIAGNÓSTICO:")
    print(f"> {analise['classificacao']}")

    print("\nATENÇÃO:")
    print(
        "Esta classificação compara a proposta com uma referência "
        "estatística. Ela NÃO determina se os juros são legais ou ilegais."
    )

    print("=" * 68)


# ============================================================
# INTERFACE
# ============================================================

def ler_float(mensagem: str) -> float:
    while True:
        try:
            valor = input(mensagem).strip().replace(".", "").replace(",", ".")
            return float(valor)
        except ValueError:
            print("Digite um número válido.")


def ler_int(mensagem: str) -> int:
    while True:
        try:
            return int(input(mensagem))
        except ValueError:
            print("Digite um número inteiro válido.")


def menu() -> None:
    print("\n" + "=" * 68)
    print(" JUROSBR — CALCULADORA E ANALISADOR DE EMPRÉSTIMOS")
    print("=" * 68)
    print("1 - Juros simples")
    print("2 - Juros compostos")
    print("3 - Analisar empréstimo")
    print("4 - Sair")

    opcao = input("\nEscolha: ").strip()

    if opcao == "1":
        principal = ler_float("Capital: R$ ")
        taxa = ler_float("Taxa por período (%): ") / 100
        periodos = ler_int("Número de períodos: ")

        juros, montante = juros_simples(
            principal,
            taxa,
            periodos,
        )

        print("\nRESULTADO")
        print(f"Juros:    {dinheiro(juros)}")
        print(f"Montante: {dinheiro(montante)}")

    elif opcao == "2":
        principal = ler_float("Capital: R$ ")
        taxa = ler_float("Taxa por período (%): ") / 100
        periodos = ler_int("Número de períodos: ")

        juros, montante = juros_compostos(
            principal,
            taxa,
            periodos,
        )

        print("\nRESULTADO")
        print(f"Juros:    {dinheiro(juros)}")
        print(f"Montante: {dinheiro(montante)}")

    elif opcao == "3":
        print("\n--- DADOS DO EMPRÉSTIMO ---")

        principal = ler_float("Valor solicitado: R$ ")
        taxa_anual = ler_float("Taxa anual (%): ") / 100
        meses = ler_int("Prazo em meses: ")
        extras = ler_float(
            "Taxas/seguros/encargos adicionais (R$ 0 se nenhum): R$ "
        )

        print("\n--- REFERÊNCIA ---")
        print(
            "Como não existe um teto único para todos os empréstimos, "
            "informe a taxa de referência que deseja utilizar."
        )

        print(
            "Exemplo: 63,0% a.a. = referência média do crédito livre "
            "para pessoa física (abr/2026)."
        )

        referencia = ler_float(
            "Taxa de referência anual (%): "
        ) / 100

        emprestimo = Emprestimo(
            principal=principal,
            taxa_anual=taxa_anual,
            meses=meses,
            taxas_extras=extras,
        )

        imprimir_relatorio(
            emprestimo,
            referencia,
        )

    elif opcao == "4":
        print("Programa encerrado.")
        return

    else:
        print("Opção inválida.")

    input("\nPressione ENTER para continuar...")
    menu()


# ============================================================
# EXECUÇÃO
# ============================================================

if __name__ == "__main__":
    menu()
