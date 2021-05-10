Google Translate Api

[Google控制台设置](https://cloud.google.com/translate/docs/setup#windows)
1. 创建项目
2. 启用结算功能
3. 启用 Cloud Translation API
4. 设置身份验证

[使用官方的C#客户端库](https://googleapis.github.io/google-cloud-dotnet/docs/Google.Cloud.Translate.V3/index.html)
1. 从NuGet安装```Google.Cloud.Translate.V3```包
2. 设置json环境变量

代码
Program.cs
```
using Google.Api.Gax.ResourceNames;
using Google.Cloud.Translate.V3;
using System;
using System.Collections.Generic;
using System.IO;
using Microsoft.Office.Interop.Excel;
using System.Linq;

class Program
{
    //string json_path = "Internel/translation-312911-98fe1180df18.json";
    //string project_id = "translation-312911";
    //string target_language_code = "ja-JP";
    //string glossary_path = "Internel/glossary.txt";
    //string input_path = @"E:\Translation\Translation\bin\Debug\Shell\input.txt";
    //string output_path = @"E:\Translation\Translation\bin\Debug\Shell\output.xlsx";

    private const int MAX_TRANSLATE_LINE = 1000; // max is 1024
    private static List<List<string>> results = new List<List<string>>();

    static void Main(string[] args)
    {
        if (args.Length < 6)
            return;

        string json_path = args[0];            // json路径
        string project_id = args[1];           // 项目ID
        string target_language_code = args[2]; // 目标语言
        string glossary_path = args[3];        // 术语表
        string input_path = args[4];           // 输入路径
        string output_path = args[5];          // 输出路径

        if (string.IsNullOrEmpty(json_path)
            || string.IsNullOrEmpty(project_id)
            || string.IsNullOrEmpty(target_language_code)
            || string.IsNullOrEmpty(glossary_path)
            || string.IsNullOrEmpty(input_path) 
            || string.IsNullOrEmpty(output_path))
        {
            return;
        }

        if (!File.Exists(input_path))
            return;

        // 设置环境变量
        Environment.SetEnvironmentVariable("GOOGLE_APPLICATION_CREDENTIALS", json_path);
        TranslationServiceClient client = TranslationServiceClient.Create();

        // 术语
        string[] glossary = { };
        if (File.Exists(glossary_path))
            glossary = File.ReadAllLines(glossary_path);

        List<string> batch_lst = new List<string>();
        List<List<string>> all_batch_lst = new List<List<string>>();

        string[] contents = File.ReadAllLines(input_path);
        for (int i = 0; i < contents.Length; i++)
        {
            if (i > 0 && i % MAX_TRANSLATE_LINE == 0)
            {
                all_batch_lst.Add(batch_lst.ToList());
                batch_lst.Clear();
            }

            string content = contents[i];
            for (int j = 0; j < glossary.Length; j++)
            {
                if (content.Contains(glossary[j]))
                    content = content.Replace(glossary[j], string.Format("<span class=\"notranslate\">{0}</span>", glossary[j]));
            }

            batch_lst.Add(content);
        }

        if (batch_lst.Count > 0)
        {
            all_batch_lst.Add(batch_lst.ToList());
            batch_lst.Clear();
        }
            
        // 每一次翻译1024行
        for (int k = 0; k < all_batch_lst.Count; k++)
        {
            TranslateTextRequest request = new TranslateTextRequest
            {
                Contents = { all_batch_lst[k] },
                TargetLanguageCode = target_language_code,
                Parent = new ProjectName(project_id).ToString(),
            };

            TranslateTextResponse response = client.TranslateText(request);
            for (int i = 0; i < response.Translations.Count; i++)
                results.Add(new List<string>() { contents[k * MAX_TRANSLATE_LINE + i], response.Translations[i].TranslatedText });
        }

        for (int i = 0; i < results.Count; i++)
        {
            for (int j = 0; j < results[i].Count; j++)
            {
                if (results[i][j].Contains("<span class=\"notranslate\">"))
                {
                    results[i][j] = results[i][j].Replace("<span class=\"notranslate\">", "");
                    results[i][j] = results[i][j].Replace("</span>", "");
                }
            }
        }

        // 输出文件
        if (File.Exists(output_path))
            File.Delete(output_path);

        ExportExcel(output_path);
    }
    
    // 导出Excel文件
    private static void ExportExcel(string output_path)
    {
        Application app = new Application();
        Workbook workbook = app.Workbooks.Add();
        Worksheet worksheet = (Worksheet)workbook.Sheets.Add();

        // 前三栏固定内容
        worksheet.Cells[1, 1] = string.Format("{0}{1}:{2},{3}:{4}{5}", "{", "\"format\"", "\"map\"", "\"key\"", "\"key\"", "}");
        worksheet.Cells[2, 1] = "auto";
        worksheet.Cells[2, 2] = "auto";
        worksheet.Cells[3, 1] = "key";
        worksheet.Cells[3, 2] = "value";

        for (int i = 0; i < results.Count; i++)
        {
            List<string> line = results[i];
            worksheet.Cells[i + 4, 1] = line[0];
            worksheet.Cells[i + 4, 2] = line[1];
        }

        app.ActiveWorkbook.SaveAs(output_path);

        workbook.Close();
        app.Quit();
    }
}
```

批处理启动脚本
start.bat
```
set JSON_PATH=Internel/translation-312911-98fe1180df18.json
set PROJECT_ID=translation-312911
set TARGET_LANGUAGE_CODE=en-US
set GLOSSARY_FILE="Internel/glossary.txt"
set INPUT_FILE=E:\Translation\Translation\bin\Debug\Shell\input.txt
set OUTPUT_FILE=E:\Translation\Translation\bin\Debug\Shell\output.xlsx

cd..
call Translation.exe %JSON_PATH% %PROJECT_ID% %TARGET_LANGUAGE_CODE% %GLOSSARY_FILE% %INPUT_FILE% %OUTPUT_FILE%
```
