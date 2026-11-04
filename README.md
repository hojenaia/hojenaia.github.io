# Marcell Freitas — blog

Blog pessoal estático em Hugo com o tema Hextra. Os artigos ficam em `content/posts/` e cada alteração na branch `main` publica o site pelo GitHub Pages.

## Desenvolvimento local

Requisitos: Hugo Extended 0.161.1 ou compatível e Go para resolver o módulo do tema.

```sh
hugo server --buildDrafts
```

## Publicação

O workflow `.github/workflows/pages.yaml` compila o site e publica no GitHub Pages a cada push em `main`. O endereço padrão é <https://hojenaia.github.io/>.
