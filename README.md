# Python: Guia Completo
![python](https://cdn-icons-png.flaticon.com/512/5968/5968286.png)

- **Autor:** *Eliseu Barbosa*
- **Data:** *15/07/2026*
- **Objetivo:** *Apresentar guia de linguagem Python.*

## Introdução:

***Python é uma linguagem de programação de alto nível, interpretada e de propósito geral, conhecida pela sua sintaxe simples e legível. Foi projetada para permitir que os programadores expressem conceitos em poucas linhas de código, ao contrário de outras linguagens mais verbosas como **C++ ou Java**. Atualmente, é uma das linguagens mais populares do mundo, usada em áreas tão diversas como desenvolvimento web, ciência de dados, inteligência artificial, automação e educação.***

## História da Linguagem:


**1989** — **Guido van Rossum**, programador holandês, começou a desenvolver o Python nos Laboratórios CWI, na Holanda, durante as férias de Natal, como um projeto pessoal para ocupar o tempo livre.

**1991** — Foi lançada a primeira versão pública, a **Python 0.9.0**, que já incluía funções, tratamento de exceções e os tipos de dados fundamentais (listas, dicionários, strings).

**2000** — Lançamento do **Python 2.0**, que introduziu recursos como list comprehensions e um sistema de recolha de lixo (garbage collector) por contagem de referências.

**2008** — Lançamento do **Python 3.0**, uma revisão importante da linguagem que não é totalmente compatível com versões anteriores, corrigindo inconsistências de design.

**2020** — Fim oficial do suporte ao Python 2, consolidando o Python 3 como a versão padrão.
O nome "Python" não vem da cobra, mas sim do grupo de comédia britânico **Monty Python**, do qual Guido van Rossum era fã.


**Hoje, o Python é mantido pela Python Software Foundation (PSF) e continua em constante evolução, com novas versões lançadas anualmente.**

## Principais Características:


 **Sintaxe simples e legível** — código próximo da linguagem natural, com indentação obrigatória a definir blocos de código.

**Interpretada** — o código é executado linha a linha por um interpretador, sem necessidade de compilação prévia.

**Tipagem dinâmica** — não é necessário declarar o tipo das variáveis.

**Multiparadigma** — suporta programação orientada a objetos, funcional e procedimental.

**Multiplataforma** — funciona em Windows, macOS, Linux, entre outros.

**Grande biblioteca padrão** — inclui módulos prontos para inúmeras tarefas ("baterias incluídas").

**Comunidade ativa e vasto ecossistema** — milhares de bibliotecas de terceiros (PyPI).
Código aberto e gratuito.


## Exemplos de Código:

```python
Olá, Mundo!

pythonprint("Olá, mundo!")

1. Variáveis e tipos de dados:

pythonnome = "Ana"
idade = 25
altura = 1.68
estudante = True

print(f"{nome} tem {idade} anos e mede {altura}m.")
```
```python
2. Estruturas de controlo:

pythonnumero = 7

if numero % 2 == 0:
    print("Número par")
else:
    print("Número ímpar")
```
```python
Ciclos (loops):

pythonfor i in range(5):
    print(f"Iteração {i}")
```
```python
Funções:

pythondef saudacao(nome):
    return f"Olá, {nome}!"

print(saudacao("Carlos"))
```
```python
Listas e list comprehension:

pythonnumeros = [1, 2, 3, 4, 5]
quadrados = [n**2 for n in numeros]
print(quadrados)  # [1, 4, 9, 16, 25]
```
```python
Programação orientada a objetos
pythonclass Pessoa:

    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        print(f"Sou {self.nome} e tenho {self.idade} anos.")

pessoa1 = Pessoa("Maria", 30)
pessoa1.apresentar()
```
![codepython](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQqX9EfELYwsloquQ5bt0UOCE-DosEXYCa6KuP9i31FtA&s=10)

## Aplicações da Linguagem:


**Desenvolvimento Web** — frameworks como [Django](https://www.djangoproject.com/) e [Flask](https://flask.palletsprojects.com/).

**Ciência de Dados e Análise de Dados** — bibliotecas como [Pandas](https://pandas.pydata.org/), [NumPy](https://numpy.org/) e [Matplotlib](https://matplotlib.org/).

**Inteligência Artificial e Machine Learning** — [TensorFlow](https://www.tensorflow.org/), [PyTorch](https://pytorch.org/), [Scikit-learn](https://scikit-learn.org/).

**Automação e Scripting** — automatização de tarefas repetitivas do sistema operativo.

**Desenvolvimento de Jogos** — [Pygame](https://www.pygame.org/).

**Internet das Coisas (IoT)** — usado com Raspberry Pi e microcontroladores.

**Educação** — muito usado no ensino de programação devido à sua simplicidade.

**Testes de Software** — automação de testes com frameworks como [Pytest](https://docs.pytest.org/).

**Cibersegurança** — scripts de análise e testes de penetração.


## Vantagens e desvantagens:


| Vantagens | Desvantagens |
|---|---|
| Sintaxe simples, ideal para iniciantes | Mais lento do que linguagens compiladas como C ou C++ |
| Elevada produtividade no desenvolvimento | Consumo de memória relativamente elevado |
| Grande quantidade de bibliotecas e frameworks | Não é a melhor opção para desenvolvimento mobile nativo |
| Comunidade enorme e bem documentada | O Global Interpreter Lock (GIL) limita o paralelismo real em threads |
| Multiplataforma | A tipagem dinâmica pode originar erros que só surgem em tempo de execução |
| Muito usado em áreas de crescimento (IA, ciência de dados) | |

## Recursos para Aprender:


- **Documentação oficial** — [python.org](python.org)
- **Plataformas de cursos** — [Coursera](https://www.coursera.org/), [Udemy](https://www.udemy.com/), [edX](https://www.edx.org/), [freeCodeCamp](https://www.freecodecamp.org/)
- **Livros** — "Automate the Boring Stuff with Python", "Python Crash Course"
- **Prática de exercícios** — [HackerRank](https://www.hackerrank.com/), [LeetCode](https://leetcode.com/), [Exercism](https://exercism.org/), [Codewars](https://www.codewars.com/)
- **Comunidades** — [Stack Overflow](https://stackoverflow.com/), [Reddit](https://www.reddit.com/) (r/learnpython).
- **Discords de programação.**
- **Canais no [YouTube](https://www.youtube.com/)** com tutoriais em português e inglês.


## Conclusão:

***Python consolidou-se como uma das linguagens de programação mais versáteis e acessíveis da atualidade. A sua simplicidade não compromete o poder, permitindo que seja usada tanto por iniciantes que dão os primeiros passos na programação como por especialistas em inteligência artificial e big data. Apesar de algumas limitações relacionadas com desempenho, as suas vantagens — comunidade ativa, vasto ecossistema de bibliotecas e curva de aprendizagem suave — tornam-na uma escolha sólida para quase qualquer tipo de projeto, sendo por isso uma excelente linguagem para aprender e dominar.***