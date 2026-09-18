# SINISA — Introdução

O **SINISA (Sistema Nacional de Informações em Saneamento Básico)** é o sistema utilizado para o registro, a consulta e a gestão de informações relacionadas ao saneamento básico.

Esta documentação descreve as funcionalidades do sistema. Cada funcionalidade deve possuir seu próprio arquivo Markdown (`.md`) neste diretório, facilitando a consulta e a manutenção do conteúdo.

## Acesso ao sistema

| Ambiente | Finalidade | Endereço |
| --- | --- | --- |
| Produção | Ambiente oficial, utilizado para a operação do sistema. | [https://sinisa.cidades.gov.br](https://sinisa.cidades.gov.br) |
| Homologação | Ambiente destinado a testes e validações antes da publicação em produção. | [https://sinisa-hmg.cidades.gov.br](https://sinisa-hmg.cidades.gov.br) |

## Organização da documentação

| Título/Link | Descrição |
| --- | --- |
| [Introdução](introducao.md) | Apresentação geral e endereços de acesso ao sistema. |
| [Dados da Coleta Anterior](dados-coleta-anterior.md) | Consulta dos dados preenchidos na referência anterior. |
| [Módulo de Gestão Municipal](gestao-municipal.md) | Descrição do módulo de gestão municipal. |
| [Casos de uso](casos-de-uso.md) | Atores e funcionalidades dos módulos de coleta. |
| [Visão geral da arquitetura](arquitetura.md) | Organização das camadas, módulos e dependências. |
| [Fluxo de login](fluxo-login.md) | Sequência de interações durante o acesso ao sistema. |

Cada nova funcionalidade deve possuir seu próprio arquivo `.md`, contendo objetivo, público envolvido, pré-requisitos e passo a passo de uso.

> Atenção: informações cadastradas no ambiente de homologação são destinadas a testes e não substituem os dados do ambiente de produção.

<p style="color: #d97706;"><strong>Nota:</strong> esta documentação ainda está em construção e será atualizada gradualmente com as demais funcionalidades do sistema.</p>
