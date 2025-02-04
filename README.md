# -Criando-um-Microsservi-o-Serverless-Valida-o-de-CPF

using System;
using System.Linq;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Logging;
using Microsoft.AspNetCore.Http;
using Newtonsoft.Json;

public static class ValidateCPFFunction
{
    // Função para validar o CPF
    private static bool ValidaCPF(string cpf)
    {
        // Remove os caracteres não numéricos
        cpf = new string(cpf.Where(char.IsDigit).ToArray());

        // Verifica se o CPF tem 11 dígitos
        if (cpf.Length != 11)
            return false;

        // Não permite CPFs como 111.111.111.11 ou 222.222.222.22, etc.
        if (cpf.All(c => c == cpf[0]))
            return false;

        // Calcula o primeiro dígito verificador
        int[] peso1 = { 10, 9, 8, 7, 6, 5, 4, 3, 2 };
        int soma1 = 0;
        for (int i = 0; i < 9; i++)
        {
            soma1 += int.Parse(cpf[i].ToString()) * peso1[i];
        }
        int resto1 = (soma1 * 10) % 11;
        if (resto1 == 10) resto1 = 0;

        // Verifica se o primeiro dígito verificador é válido
        if (resto1 != int.Parse(cpf[9].ToString()))
            return false;

        // Calcula o segundo dígito verificador
        int[] peso2 = { 11, 10, 9, 8, 7, 6, 5, 4, 3, 2 };
        int soma2 = 0;
        for (int i = 0; i < 10; i++)
        {
            soma2 += int.Parse(cpf[i].ToString()) * peso2[i];
        }
        int resto2 = (soma2 * 10) % 11;
        if (resto2 == 10) resto2 = 0;

        // Verifica se o segundo dígito verificador é válido
        return resto2 == int.Parse(cpf[10].ToString());
    }

    // Função principal da Azure Function
    [FunctionName("ValidateCPF")]
    public static async Task<HttpResponseMessage> Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequestMessage req,
        ILogger log)
    {
        // Obtém o CPF da query string ou do corpo da requisição
        var queryString = req.GetQueryNameValuePairs();
        var cpf = queryString.FirstOrDefault(q => string.Compare(q.Key, "cpf", true) == 0).Value;

        if (string.IsNullOrEmpty(cpf))
        {
            var content = await req.Content.ReadAsStringAsync();
            dynamic data = JsonConvert.DeserializeObject(content);
            cpf = data?.cpf;
        }

        if (string.IsNullOrEmpty(cpf))
        {
            return req.CreateResponse(HttpStatusCode.BadRequest, "Por favor, forneça um CPF.");
        }

        // Valida o CPF
        if (ValidaCPF(cpf))
        {
            return req.CreateResponse(HttpStatusCode.OK, $"CPF {cpf} é válido.");
        }
        else
        {
            return req.CreateResponse(HttpStatusCode.BadRequest, $"CPF {cpf} é inválido.");
        }
    }
}
