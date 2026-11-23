
"""
RentSmart - Sistema de Orçamento de Locação da R.M Imóveis
Aplicação em Python com Programação Orientada a Objetos.

Estrutura de classes:
    Imovel (base)  ->  Apartamento, Casa, Estudio
    Orcamento      ->  junta imóvel + taxa contratual + parcelamento
    main()         ->  interface de terminal (entrada e validação)
"""

import csv

TAXA_CONTRATUAL = 2000.00
MAX_PARCELAS = 5
MESES_PROJECAO = 12


def moeda(valor):
    """Formata um número como moeda brasileira: 1234.5 -> R$ 1.234,50"""
    texto = f"{valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", ".")
    return f"R$ {texto}"


# ---------------------------------------------------------------------------
# IMÓVEIS
# ---------------------------------------------------------------------------
class Imovel:
    """Classe base: guarda o que é comum a todo imóvel e define o 'contrato'
    que cada tipo deve cumprir (polimorfismo)."""

    nome = "Imóvel"
    aluguel_base = 0.0

    def __init__(self, quartos, vagas):
        self.quartos = quartos
        self.vagas = vagas
        self.validar()

    def validar(self):
        """Cada subclasse define suas regras. Lança ValueError se inválido."""
        raise NotImplementedError

    def adicionais(self):
        """Retorna lista de (descrição, valor) dos adicionais do aluguel."""
        raise NotImplementedError

    def desconto(self):
        """Retorna (descrição, valor) do desconto. Padrão: sem desconto."""
        return ("Sem desconto", 0.0)

    def total_adicionais(self):
        return sum(valor for _, valor in self.adicionais())

    def aluguel_antes_desconto(self):
        return self.aluguel_base + self.total_adicionais()

    def valor_mensal(self):
        return round(self.aluguel_antes_desconto() - self.desconto()[1], 2)


class Apartamento(Imovel):
    nome = "Apartamento"
    aluguel_base = 700.00
    ADICIONAL_2_QUARTOS = 200.00
    VALOR_GARAGEM = 300.00
    DESCONTO_SEM_CRIANCAS = 0.05

    def __init__(self, quartos, vagas, tem_criancas):
        self.tem_criancas = tem_criancas
        super().__init__(quartos, vagas)

    def validar(self):
        if self.quartos not in (1, 2):
            raise ValueError("Apartamento deve ter 1 ou 2 quartos.")
        if self.vagas not in (0, 1):
            raise ValueError("Apartamento aceita 0 ou 1 vaga de garagem.")

    def adicionais(self):
        itens = []
        if self.quartos == 2:
            itens.append(("Adicional 2º quarto", self.ADICIONAL_2_QUARTOS))
        if self.vagas == 1:
            itens.append(("Vaga de garagem", self.VALOR_GARAGEM))
        return itens

    def desconto(self):
        # 5% sobre o aluguel já com os adicionais, só se NÃO tiver crianças
        if not self.tem_criancas:
            valor = round(self.aluguel_antes_desconto() * self.DESCONTO_SEM_CRIANCAS, 2)
            return ("Desconto 5% (sem crianças)", valor)
        return super().desconto()


class Casa(Imovel):
    nome = "Casa"
    aluguel_base = 900.00
    ADICIONAL_2_QUARTOS = 250.00
    VALOR_GARAGEM = 300.00

    def validar(self):
        if self.quartos not in (1, 2):
            raise ValueError("Casa deve ter 1 ou 2 quartos.")
        if self.vagas not in (0, 1):
            raise ValueError("Casa aceita 0 ou 1 vaga de garagem.")

    def adicionais(self):
        itens = []
        if self.quartos == 2:
            itens.append(("Adicional 2º quarto", self.ADICIONAL_2_QUARTOS))
        if self.vagas == 1:
            itens.append(("Vaga de garagem", self.VALOR_GARAGEM))
        return itens


class Estudio(Imovel):
    nome = "Estúdio"
    aluguel_base = 1200.00
    PACOTE_INICIAL_VAGAS = 2
    VALOR_PACOTE = 250.00
    VALOR_VAGA_EXTRA = 60.00

    def __init__(self, vagas):
        super().__init__(quartos=0, vagas=vagas)

    def validar(self):
        if self.vagas < 0:
            raise ValueError("Número de vagas não pode ser negativo.")

    def adicionais(self):
        itens = []
        if self.vagas >= 1:
            # Quem contrata estacionamento paga o pacote de 2 vagas
            itens.append((f"Pacote de {self.PACOTE_INICIAL_VAGAS} vagas", self.VALOR_PACOTE))
            extras = self.vagas - self.PACOTE_INICIAL_VAGAS
            if extras > 0:
                itens.append((f"{extras} vaga(s) adicional(is) x {moeda(self.VALOR_VAGA_EXTRA)}",
                              extras * self.VALOR_VAGA_EXTRA))
        return itens


# ---------------------------------------------------------------------------
# ORÇAMENTO
# ---------------------------------------------------------------------------
class Orcamento:
    """Une o imóvel escolhido, a taxa contratual e o parcelamento."""

    def __init__(self, cliente, imovel, parcelas_taxa):
        if not 1 <= parcelas_taxa <= MAX_PARCELAS:
            raise ValueError(f"A taxa pode ser paga em 1 a {MAX_PARCELAS} parcelas.")
        self.cliente = cliente
        self.imovel = imovel
        self.parcelas_taxa = parcelas_taxa
        self.taxa = TAXA_CONTRATUAL

    def valores_parcelas(self):
        """Divide a taxa; a última parcela absorve o centavo de arredondamento."""
        parcela = round(self.taxa / self.parcelas_taxa, 2)
        lista = [parcela] * self.parcelas_taxa
        lista[-1] = round(self.taxa - parcela * (self.parcelas_taxa - 1), 2)
        return lista

    def resumo(self):
        im = self.imovel
        linhas = [
            "=" * 52,
            "        RENTSMART - ORÇAMENTO DE LOCAÇÃO",
            "=" * 52,
            f"Cliente: {self.cliente}",
            f"Imóvel : {im.nome}" + (f" - {im.quartos} quarto(s)" if im.quartos else ""),
            "",
            "--- ALUGUEL MENSAL ---",
            f"Aluguel-base ................ {moeda(im.aluguel_base):>14}",
        ]
        for descricao, valor in im.adicionais():
            linhas.append(f"+ {descricao:.<26} {moeda(valor):>14}")
        linhas.append(f"Subtotal .................... {moeda(im.aluguel_antes_desconto()):>14}")
        desc, valor_desc = im.desconto()
        if valor_desc > 0:
            linhas.append(f"- {desc:.<26} {moeda(valor_desc):>14}")
        linhas.append(f"VALOR MENSAL ................ {moeda(im.valor_mensal()):>14}")
        linhas += [
            "",
            "--- TAXA CONTRATUAL (cobrada à parte) ---",
            f"Valor da taxa ............... {moeda(self.taxa):>14}",
            f"Parcelamento ................ {self.parcelas_taxa:>13}x",
            f"Valor por parcela ........... {moeda(self.valores_parcelas()[0]):>14}",
            "=" * 52,
        ]
        return "\n".join(linhas)

    def projecao_12_meses(self):
        """Gera uma linha por mês: aluguel + parcela da taxa (se houver)."""
        parcelas = self.valores_parcelas()
        linhas = []
        acumulado = 0.0
        for mes in range(1, MESES_PROJECAO + 1):
            aluguel = self.imovel.valor_mensal()
            taxa_mes = parcelas[mes - 1] if mes <= self.parcelas_taxa else 0.0
            total = round(aluguel + taxa_mes, 2)
            acumulado = round(acumulado + total, 2)
            linhas.append({
                "Mês": mes,
                "Aluguel": aluguel,
                "Parcela da taxa contratual": taxa_mes,
                "Total do mês": total,
                "Total acumulado": acumulado,
            })
        return linhas

    def exportar_csv(self, caminho="projecao_12_meses.csv"):
        linhas = self.projecao_12_meses()
        # utf-8-sig e ';' para abrir corretamente no Excel em português
        with open(caminho, "w", newline="", encoding="utf-8-sig") as arq:
            escritor = csv.writer(arq, delimiter=";")
            escritor.writerow(linhas[0].keys())
            for linha in linhas:
                escritor.writerow([
                    linha["Mês"],
                    *(f"{v:.2f}".replace(".", ",") for k, v in linha.items() if k != "Mês"),
                ])
        return caminho


# ---------------------------------------------------------------------------
# ENTRADA DE DADOS COM VALIDAÇÃO
# ---------------------------------------------------------------------------
def ler_inteiro(pergunta, validos):
    """Repete a pergunta até o usuário digitar um inteiro dentro de 'validos'."""
    while True:
        texto = input(pergunta).strip()
        if texto.isdigit() and int(texto) in validos:
            return int(texto)
        print(f"  Entrada inválida. Opções permitidas: {', '.join(map(str, validos))}.")


def ler_sim_nao(pergunta):
    while True:
        texto = input(pergunta + " (s/n): ").strip().lower()
        if texto in ("s", "sim"):
            return True
        if texto in ("n", "nao", "não"):
            return False
        print("  Responda com 's' ou 'n'.")


def ler_nome():
    while True:
        nome = input("Nome do cliente: ").strip()
        if len(nome) >= 2:
            return nome
        print("  Digite um nome válido.")


def montar_imovel():
    print("\nTipo de imóvel:\n  1 - Apartamento\n  2 - Casa\n  3 - Estúdio")
    tipo = ler_inteiro("Escolha: ", (1, 2, 3))

    if tipo == 3:
        print("\nEstúdio: pacote inicial de 2 vagas (R$ 250,00) + R$ 60,00 por vaga extra.")
        vagas = ler_inteiro("Quantas vagas deseja (0 a 10)? ", range(0, 11))
        return Estudio(vagas)

    quartos = ler_inteiro("Quantidade de quartos (1 ou 2): ", (1, 2))
    vagas = 1 if ler_sim_nao("Deseja vaga de garagem?") else 0
    if tipo == 1:
        tem_criancas = ler_sim_nao("Possui crianças?")
        return Apartamento(quartos, vagas, tem_criancas)
    return Casa(quartos, vagas)


def main():
    print("=== RentSmart - R.M Imóveis ===\n")
    cliente = ler_nome()
    imovel = montar_imovel()
    parcelas = ler_inteiro(f"\nTaxa contratual de {moeda(TAXA_CONTRATUAL)}: "
                           f"parcelar em quantas vezes (1 a {MAX_PARCELAS})? ",
                           range(1, MAX_PARCELAS + 1))

    orcamento = Orcamento(cliente, imovel, parcelas)
    print("\n" + orcamento.resumo())

    arquivo = orcamento.exportar_csv()
    print(f"\nProjeção de {MESES_PROJECAO} meses salva em: {arquivo}")


if __name__ == "__main__":
    main()


    # RentSmart — Sistema de Orçamento de Locação (R.M Imóveis)

Aplicação em Python (Programação Orientada a Objetos) que recebe as características da locação, aplica as regras comerciais da R.M Imóveis, calcula o aluguel mensal, mostra a taxa contratual separadamente e gera um CSV com a projeção dos 12 primeiros meses.

## Como executar

Requisitos: Python 3.8 ou superior (usa apenas a biblioteca padrão, não precisa instalar nada).

```bash
python rentsmart.py
```

O programa pergunta nome do cliente, tipo de imóvel, quartos, garagem, crianças (apenas apartamento) e parcelamento da taxa. Ao final, exibe o orçamento e cria o arquivo `projecao_12_meses.csv` na mesma pasta.

## Regras implementadas

| Item | Regra |
|---|---|
| Apartamento 1 quarto | R$ 700,00 (2 quartos: +R$ 200,00) |
| Casa 1 quarto | R$ 900,00 (2 quartos: +R$ 250,00) |
| Estúdio | R$ 1.200,00 |
| Garagem (casa/apto) | +R$ 300,00 (1 vaga) |
| Estacionamento estúdio | Pacote de 2 vagas R$ 250,00 + R$ 60,00 por vaga extra |
| Desconto | 5% para apartamento sem crianças, sobre o aluguel já com adicionais |
| Taxa contratual | R$ 2.000,00 em 1 a 5 parcelas, cobrada separada do aluguel |

### Decisões de interpretação
- O desconto de 5% incide sobre base + adicionais (2º quarto e garagem).
- No estúdio, o estacionamento é opcional; se contratado, vale o pacote de 2 vagas, com extras a R$ 60,00 cada.
- A taxa é dividida igualmente; a última parcela absorve a diferença de centavos.
- A taxa é paga nos primeiros meses (uma parcela por mês) e aparece em coluna própria no CSV.

## Estrutura de classes

- `Imovel` (classe base): atributos `quartos`, `vagas`; métodos `adicionais()`, `desconto()`, `valor_mensal()`.
- `Apartamento`, `Casa`, `Estudio`: herdam de `Imovel` e implementam suas próprias regras (polimorfismo).
- `Orcamento`: agrega imóvel + taxa + parcelas; métodos `resumo()`, `valores_parcelas()`, `projecao_12_meses()`, `exportar_csv()`.

## Arquivo CSV
Separador `;` e codificação UTF-8 com BOM (abre direto no Excel em português). Colunas: Mês, Aluguel, Parcela da taxa contratual, Total do mês, Total acumulado.

## Exemplo de saída
Veja `projecao_12_meses.csv` (apartamento 2 quartos, com garagem, sem crianças, taxa em 3x).
Mês;Aluguel;Parcela da taxa contratual;Total do mês;Total acumulado
1;1570,00;400,00;1970,00;1970,00
2;1570,00;400,00;1970,00;3940,00
3;1570,00;400,00;1970,00;5910,00
4;1570,00;400,00;1970,00;7880,00
5;1570,00;400,00;1970,00;9850,00
6;1570,00;0,00;1570,00;11420,00
7;1570,00;0,00;1570,00;12990,00
8;1570,00;0,00;1570,00;14560,00
9;1570,00;0,00;1570,00;16130,00
10;1570,00;0,00;1570,00;17700,00
11;1570,00;0,00;1570,00;19270,00
12;1570,00;0,00;1570,00;20840,00
