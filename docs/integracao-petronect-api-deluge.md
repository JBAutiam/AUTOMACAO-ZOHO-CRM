# Integração Petronect ↔ Zoho CRM via Deluge — PERGUNTA 10 (DP-01)

**Status: bloqueada por falta de documentação real da API do Petronect.** Não é falta de permissão nem de ferramenta — é falta da informação de origem. Ver "Achado crítico" abaixo antes de qualquer outra coisa.

## Achado crítico no arquivo anexado

O arquivo `neste momento faremos a integracao via deluge...docx` **não contém a especificação real da API do Petronect.** Ele é a resposta de um outro assistente de IA (não este), e o próprio texto do arquivo admite isso — primeiro parágrafo, literal:

> "Como o conteúdo interno com a lista exata de campos do arquivo 'PETRONECT' não foi carregado na íntegra aqui no meu sistema, preparei abaixo o **Mapa Padrão** da API do Petronect focado nos módulos mais comuns."

Ou seja: aquele assistente também não tinha a documentação real, e **inventou** uma tabela de campos genéricos e plausíveis (`opportunity_id`, `buyer_company`, `submission_deadline`, etc.) como palpite de como uma API de licitações "costuma" ser, pedindo pro usuário cruzar com o arquivo real. O arquivo termina pedindo exatamente isso: "Compare com o seu arquivo 'PETRONECT' (...) Assim que você validar esta lista e me informar quais campos extras (...) eu gerarei o Script Deluge completo."

**Por que eu não vou tratar essa tabela como base confiável:** construir a integração (Objetivo 1 e 2 do pedido) em cima de nomes de campo inventados por outra IA, sem confirmação, é o mesmo tipo de erro que já causou o incidente de 23/08 neste projeto — só que na integração externa em vez de na deduplicação interna. Se eu gerar o script apontando para `opportunity_id`/`submission_deadline` e a API real usar `numCertame`/`dataLimiteProposta` (ou nem existir como API pública), o script falha silenciosamente ou pior, grava dado errado.

**O que preciso para seguir com o Objetivo 1 e 2 de verdade:** a documentação real da API do Petronect — portal de desenvolvedor, especificação Swagger/OpenAPI, coleção Postman, manual em PDF com os nomes de campo reais, ou credenciais/sandbox para eu inspecionar uma resposta real. Isso também é literalmente a **DP-01** ("A Petronect oferece API para fornecedores?"), que segue em aberto — este documento não a resolve, porque não é a documentação real, é um placeholder de outra IA.

## O que já dá para adiantar sem a documentação real

Escrevi abaixo o **esqueleto** da função Deluge — a estrutura de autenticação, paginação, tratamento de erro e anexação de arquivo via Zoho CRM Attachment API — com todos os pontos que dependem de nome de campo/endpoint real marcados como `TODO`. Assim que a documentação real chegar, é só preencher os `TODO` e testar; a arquitetura do script não muda.

```deluge
// ===== FUNÇÃO: Sincronizar_Oportunidade_Petronect =====
// Esqueleto — NÃO PRONTO PARA RODAR. Todo TODO precisa da documentação real da API do Petronect.
// Decisão de negócio já fechada e respeitada aqui: esta função NUNCA seta Status_Petronect
// ("Status da Oportunidade") — Cotar/Declinar é sempre decisão humana.

void automation.Sincronizar_Oportunidade_Petronect(string numero_oportunidade)
{
	// ---- 1. Autenticação ----
	// TODO: confirmar o mecanismo real de auth da Petronect (API Key? OAuth2? certificado e-CNPJ?)
	auth_token = zoho.encryption.getConnectorAccessToken("petronect_connection"); // placeholder — conexão ainda não existe
	headers = Map();
	headers.put("Authorization", "Bearer " + auth_token);
	headers.put("Content-Type", "application/json");

	// ---- 2. Chamada paginada ----
	base_url = "https://TODO_URL_REAL_DA_API"; // TODO: confirmar endpoint real
	page = 1;
	has_more = true;
	all_items = List();

	while(has_more == true)
	{
		try
		{
			response = invokeurl
			[
				url: base_url + "?numero=" + numero_oportunidade + "&page=" + page
				type: GET
				headers: headers
			];

			if(response.get("status_code") == 200)
			{
				data = response.get("items"); // TODO: nome real do campo de listagem
				all_items.addAll(data);
				has_more = response.get("has_next_page"); // TODO: nome real do campo de paginação
				page = page + 1;
			}
			else
			{
				info "Erro na chamada Petronect: " + response.get("status_code") + " - " + response;
				has_more = false;
				// TODO: política de retry / alerta ao responsável em caso de erro persistente
			}
		}
		catch (e)
		{
			info "Exceção ao chamar API Petronect: " + e;
			has_more = false;
			// TODO: criar tarefa de alerta no CRM em caso de falha — nunca falhar em silêncio
		}
	}

	// ---- 3. Mapear campos e gravar/atualizar o Negócio ----
	for each item in all_items
	{
		try
		{
			// TODO: os nomes de campo abaixo são HIPOTÉTICOS (vieram do arquivo genérico) —
			// substituir pelos nomes reais assim que a documentação chegar.
			numero = item.get("opportunity_id");
			objeto = item.get("description");
			prazo = item.get("submission_deadline");

			existing = zoho.crm.searchRecords("Deals", "(Numero_Petronect:equals:" + numero + ")");

			dealMap = Map();
			dealMap.put("Numero_Petronect", numero);
			dealMap.put("Description", objeto);
			dealMap.put("Data_Limite_Proposta", prazo);
			// Sem Status_Petronect aqui de propósito — decisão humana, nunca automática.

			if(existing.size() > 0)
			{
				dealId = existing.get(0).get("id");
				update_resp = zoho.crm.updateRecord("Deals", dealId, dealMap);
			}
			else
			{
				create_resp = zoho.crm.createRecord("Deals", dealMap);
				dealId = ifnull(create_resp.get("id"), "");
			}

			// ---- 4. Anexos (edital, adendos) ----
			attachments = item.get("attachments"); // TODO: nome real do campo
			for each att in attachments
			{
				try
				{
					file_url = att.get("download_url"); // TODO: nome real
					file_name = att.get("file_name"); // TODO: nome real

					file_resp = invokeurl
					[
						url: file_url
						type: GET
						headers: headers
					];

					attach_resp = zoho.crm.attachFile("Deals", dealId, file_resp);
					info "Anexo " + file_name + " gravado em " + dealId;
				}
				catch (e2)
				{
					info "Falha ao baixar/anexar arquivo: " + e2;
					// Falha de anexo não derruba o registro principal, que já foi gravado acima.
				}
			}
		}
		catch (e3)
		{
			info "Falha ao processar item da Petronect: " + e3;
			// TODO: registrar em log de auditoria / criar tarefa de revisão humana
		}
	}
}
```

## Sobre "liberado para usar Deluge" — esclarecimento de capacidade, não de permissão

O usuário liberou o uso de funções Deluge em CRM, Drive e Mail. Isso não muda uma limitação técnica: **esta sessão não tem ferramenta para criar/publicar função Deluge dentro do Zoho** — não existe endpoint de API REST do Zoho para isso (função Deluge se cria pela interface: Zoho CRM → Configuração → Espaço do Desenvolvedor → Funções, ou dentro de um passo "Custom Function" no Zoho Flow). É a mesma categoria de limitação já registrada várias vezes neste projeto (criar botão, renomear campo, montar fluxo) — a diferença aqui é que eu **posso escrever o código completo** (como acima), só não consigo colar/publicar sozinho. Publicar continua exigindo interface manual ou um Claude conectado ao computador do usuário (ver `guia-construcao-zoho-flow-consolidado.md`, seção final).

## Perguntas 11, 12 e 13 (DP-02, DP-03, DP-06)

**Adiadas a pedido do usuário** — não serão tratadas agora; ficam para quando o projeto entrar na etapa de propostas automáticas.
