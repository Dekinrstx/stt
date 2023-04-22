using System;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;
using Newtonsoft.Json;
using Telegram.Bot;
using Telegram.Bot.Args;
using Telegram.Bot.Types;
using Telegram.Bot.Types.Enums;
using Telegram.Bot.Types.ReplyMarkups;

namespace TelegramBot
{
    public static class Program
    {
        private static readonly HttpClient client = new HttpClient();
        private static readonly string token = "6200286187:AAHAxCKHw4w1TyGJWiLNhxUbPW_JZCiEEkc";
        private static readonly TelegramBotClient bot = new TelegramBotClient(token);

        public static async Task Main()
        {
            bot.OnMessage += Bot_OnMessage;
            bot.StartReceiving();

            Console.WriteLine("хуй пизда залупа");
            Console.ReadKey();

            bot.StopReceiving();
        }

        private static async void Bot_OnMessage(object sender, MessageEventArgs e)
        {
            var message = e.Message;

            if (message.Type == MessageType.Text)
            {
                switch (message.Text)
                {
                    case "/start":
                        await ShowMainMenu(message);
                        break;
                    case "FAQ":
                        await ShowFaq(message);
                        break;
                    case "О боте":
                        await ShowAbout(message);
                        break;
                    case "Проверка криптокошельков":
                        await CheckWallets(message);
                        break;
                }
            }
            else if (message.Type == MessageType.Document)
            {
                await ProcessWalletsFile(message);
            }
        }

        private static async Task ShowMainMenu(Message message)
        {
            var keyboard = new ReplyKeyboardMarkup(new[]
            {
        new KeyboardButton("FAQ"),
        new KeyboardButton("О боте"),
        new KeyboardButton("Проверка криптокошельков")
    }, resizeKeyboard: true);

            await bot.SendTextMessageAsync(message.Chat.Id, "Выберите действие:", replyMarkup: keyboard);
        }

        private static async Task ShowFaq(Message message)
        {
            await bot.SendTextMessageAsync(message.Chat.Id, "Контакты поддержки: @user");
        }

        private static async Task ShowAbout(Message message)
        {
            await bot.SendTextMessageAsync(message.Chat.Id, "Этот бот позволяет проверять балансы криптокошельков с помощью API Zerion.");
        }

        private static async Task CheckWallets(Message message)
        {
            await bot.SendTextMessageAsync(message.Chat.Id, "Пожалуйста, отправьте файл с криптокошельками (максимум 20 МБ).");
        }

        private static async Task ProcessWalletsFile(Message message)
        {
            var file = await bot.GetFileAsync(message.Document.FileId);
            using var stream = new MemoryStream();
            await bot.DownloadFileAsync(file.FilePath, stream);
            stream.Position = 0;

            using var reader = new StreamReader(stream);
            var content = await reader.ReadToEndAsync();
            var wallets = content.Split(new[] { "\r\n", "\r", "\n" }, StringSplitOptions.RemoveEmptyEntries);

            foreach (var wallet in wallets)
            {
                var result = await CheckWalletBalance(wallet);
                await bot.SendTextMessageAsync(message.Chat.Id, result, ParseMode.Markdown);
            }
        }

        private static async Task<string> CheckWalletBalance(string wallet)
        {
            try
            {
                client.DefaultRequestHeaders.Accept.Clear();
                client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
                client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", "emtfZGV2XzkxMDc3NjM4YTFiYjRkYmRhYmNhZGM4ZDExN2EwNjM0Og==");

                var portfolioUrl = $"https://api.zerion.io/v1/wallets/{wallet}/portfolio?currency=usd";
                Console.WriteLine($"Sending request to {portfolioUrl}");
                var portfolioResponse = await client.GetAsync(portfolioUrl);
                portfolioResponse.EnsureSuccessStatusCode();

                var portfolioJson = await portfolioResponse.Content.ReadAsStringAsync();
                var portfolioData = JsonConvert.DeserializeObject<ZerionResponse>(portfolioJson);

                var totalBalance = portfolioData.Data.Attributes.Total.Positions;

                var result = $"*Wallet:* {wallet}\n*Total balance:* {totalBalance:0.00} USD\n\n*Coins:*";

                foreach (var coin in portfolioData.Data.Attributes.PositionsDistributionByChain)
                {
                    if (coin.Value > 0)
                    {
                        result += $"\n- {coin.Key}: {coin.Value:0.00} USD";
                    }
                }

                var nftUrl = $"https://api.zerion.io/v1/wallets/{wallet}/nft-positions/?currency=usd&page[size]=100";
                Console.WriteLine($"Sending request to {nftUrl}");
                var nftResponse = await client.GetAsync(nftUrl);
                nftResponse.EnsureSuccessStatusCode();

                var nftJson = await nftResponse.Content.ReadAsStringAsync();
                var nftData = JsonConvert.DeserializeObject<ZerionNftResponse>(nftJson);

                result += "\n\n*NFTs:*";

                foreach (var nft in nftData.Data)
                {
                    result += $"\n- {nft.Attributes.Name}: {nft.Attributes.Value:0.00} USD";
                }

                return result;
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.ToString());
                return $"Error checking wallet {wallet}: {ex.Message}";
            }
        }


        public class ZerionResponse
        {
            [JsonProperty("links")]
            public Links Links { get; set; }

            [JsonProperty("data")]
            public Data Data { get; set; }
        }

        public class Links
        {
            [JsonProperty("self")]
            public string Self { get; set; }
        }

        public class Data
        {
            [JsonProperty("type")]
            public string Type { get; set; }

            [JsonProperty("id")]
            public string Id { get; set; }

            [JsonProperty("attributes")]
            public Attributes Attributes { get; set; }
        }

        public class Attributes
        {
            [JsonProperty("positions_distribution_by_type")]
            public Dictionary<string, double> PositionsDistributionByType { get; set; }

            [JsonProperty("positions_distribution_by_chain")]
            public Dictionary<string, double> PositionsDistributionByChain { get; set; }

            [JsonProperty("total")]
            public Total Total { get; set; }

            [JsonProperty("changes")]
            public Changes Changes { get; set; }
        }

        public class Total
        {
            [JsonProperty("positions")]
            public double Positions { get; set; }
        }

        public class Changes
        {
            [JsonProperty("absolute_1d")]
            public double? Absolute1d { get; set; }

            [JsonProperty("percent_1d")]
            public double? Percent1d { get; set; }
        }
        public class ZerionNftResponse
        {
            [JsonProperty("data")]
            public NftData[] Data { get; set; }
        }

        public class NftData
        {
            [JsonProperty("attributes")]
            public NftAttributes Attributes { get; set; }
        }

        public class NftAttributes
        {
            [JsonProperty("name")]
            public string Name { get; set; }

            [JsonProperty("value")]
            public double Value { get; set; }
        }
    }
}
