# Documentação do SINISA

Portal de documentação técnica e funcional dos sistemas SINISA Locais e
SINISA Regionais, gerado com [Zensical](https://zensical.org/).

## Executar localmente no Windows

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
zensical serve
```

Depois, acesse `http://localhost:8000`.

## Validar a documentação

```powershell
zensical build --strict --clean
```

O site estático será gerado em `site/`. Os arquivos-fonte ficam em `docs/`:

- [SINISA Locais](docs/sinisa-locais/introducao.md)
- [SINISA Regionais](docs/sinisa-regionais/introducao.md)
- Aspectos de desenvolvimento: [Xdebug](docs/aspectos-desenvolvimento/configurando-xdebug.md) e [Gii](docs/aspectos-desenvolvimento/construcao-modulos-gii.md).
- [Registros de decisões arquiteturais](docs/adr/)


<sub>Obs.: teste de migração de owner</sub>