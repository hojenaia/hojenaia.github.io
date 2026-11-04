# Marcell Freitas — blog

Blog pessoal estático em Hugo com o tema Hextra. Os artigos ficam em `content/posts/` e cada alteração na branch `main` publica o site pelo GitHub Pages.

## Desenvolvimento local

Requisitos: Hugo Extended 0.161.1 ou compatível e Go para resolver o módulo do tema.

```sh
hugo server --buildDrafts
```

## Publicação

O workflow `.github/workflows/pages.yaml` compila o site e publica no GitHub Pages a cada push em `main`. O endereço padrão é <https://hojenaia.github.io/>.

## Conteúdo em dois idiomas

O português é o idioma padrão do site. As páginas e os artigos em inglês ficam em `content/en/` e aparecem sob `/en/`; mantenha o mesmo caminho relativo do conteúdo em português para que o seletor de idioma encontre a página equivalente.

Os layouts personalizados ficam em `layouts/`, enquanto estilos e interações do site ficam em `static/`.
