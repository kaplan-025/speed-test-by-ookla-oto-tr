# speed-test-by-ookla-oto-tr
Speed ​​test by ookla allows you to perform speed tests on all servers in Türkiye.

```markdown
// Türkiye'deki 81 ilin adı, küçük harfler ve Türkçe karakterler olmadan listelenmiştir.
// İstenirse bu listeye ilçe adları da eklenebilir.
const turkishCities = [
  "adana", "adiyaman", "afyonkarahisar", "agri", "aksaray", "amasya", "ankara",
  "antalya", "ardahan", "artvin", "aydin", "balikesir", "bartin", "batman",
  "bayburt", "bilecik", "bingol", "bitlis", "bolu", "burdur", "bursa",
  "canakkale", "cankiri", "corum", "denizli", "diyarbakir", "duzce", "edirne",
  "elazig", "erzincan", "erzurum", "eskisehir", "gaziantep", "giresun",
  "gumushane", "hakkari", "hatay", "igdir", "isparta", "istanbul", "izmir",
  "kahramanmaras", "karabuk", "karaman", "kars", "kastamonu", "kayseri", "kilis",
  "kirikkale", "kirklareli", "kirsehir", "kocaeli", "konya", "kutahya", "malatya",
  "manisa", "mardin", "mersin", "mugla", "mus", "nevsehir", "nigde", "ordu",
  "osmaniye", "rize", "sakarya", "samsun", "sanliurfa", "siirt", "sinop",
  "sirnak", "sivas", "tekirdag", "tokat", "trabzon", "tunceli", "usak", "van",
  "yalova", "yozgat", "zonguldak"
];

// Belirli bir elementin sayfada görünmesini bekleyen yardımcı fonksiyon.
// İstenilen süre içinde elementi bulup etkileşime hazır olana kadar bekler.
async function waitForElement(selector, timeout = 30000) {
  console.log(`[${new Date().toLocaleTimeString()}] Eleman aranıyor: ${selector}`);
  const startTime = Date.now();
  while (Date.now() - startTime < timeout) {
    const elements = document.querySelectorAll(selector);
    for (const element of elements) {
      // Elementin varlığını, görünürlüğünü ve etkileşime hazır olup olmadığını kontrol et
      if (element && element.offsetParent !== null && !element.disabled) {
        console.log(`[${new Date().toLocaleTimeString()}] Eleman bulundu ve hazır: ${selector}`);
        return element;
      }
    }
    await new Promise(resolve => setTimeout(resolve, 500));
  }
  console.error(`[${new Date().toLocaleTimeString()}] Hata: Zaman aşımı. Element bulunamadı veya etkileşime hazır olmadı: ${selector}`);
  return null;
}

// Sunucu listesinin belirli bir şehir için güncellenmesini bekleyen fonksiyon.
// Arama sonucunda listenin dolmasını veya güncellenmesini sağlar.
async function waitForServerListUpdate(city, containerSelector, timeout = 15000) {
  const startTime = Date.now();
  console.log(`[${new Date().toLocaleTimeString()}] Sunucu listesinin "${city}" için güncellenmesi bekleniyor...`);
  while (Date.now() - startTime < timeout) {
    const container = document.querySelector(containerSelector);
    if (container) {
      const serverItems = container.querySelectorAll('li');
      // En az bir elemanın şehir adını içerdiğini kontrol et
      const listUpdated = Array.from(serverItems).some(item => item.textContent.toLowerCase().includes(city.toLowerCase()));
      if (listUpdated) {
        console.log(`[${new Date().toLocaleTimeString()}] Sunucu listesi "${city}" için güncellendi.`);
        return Array.from(serverItems).map(item => item.textContent.trim());
      }
    }
    await new Promise(resolve => setTimeout(resolve, 500));
  }
  console.warn(`[${new Date().toLocaleTimeString()}] Uyarı: Zaman aşımı. Sunucu listesi "${city}" için güncellenemedi.`);
  return [];
}

// Seçilen sunucunun ana sayfada göründüğünü doğrulayan yeni yardımcı fonksiyon
async function waitForServerChange(serverName, city, timeout = 10000) {
  console.log(`[${new Date().toLocaleTimeString()}] Ana sayfada sunucu değişiminin "${serverName}" olarak tamamlanması bekleniyor...`);
  const startTime = Date.now();
  const currentServerSelector = '.server-current .result-data .name';
  while (Date.now() - startTime < timeout) {
    const serverElement = document.querySelector(currentServerSelector);
    if (serverElement) {
      const currentText = serverElement.textContent.trim().toLowerCase();
      // Sunucunun ana sayfa isminde şehrin geçip geçmediğini kontrol et
      if (currentText.includes(city.toLowerCase())) {
        console.log(`[${new Date().toLocaleTimeString()}] Sunucu başarıyla "${serverName}" olarak değiştirildi.`);
        return true;
      }
    }
    await new Promise(resolve => setTimeout(resolve, 500));
  }
  console.error(`[${new Date().toLocaleTimeString()}] Hata: Zaman aşımı. Sunucu "${serverName}" olarak değiştirilemedi.`);
  return false;
}

// Metni belirli bir elemente kademeli olarak yazıp olayları tetikleyen fonksiyon.
// İnsan benzeri bir yazma deneyimi oluşturur.
async function typeTextPartially(element, text) {
  const halfLength = Math.ceil(text.length / 2);
  const firstPart = text.substring(0, halfLength);
  const secondPart = text.substring(halfLength);

  // İlk kısmı yaz
  element.value = firstPart;
  element.dispatchEvent(new Event('input', { bubbles: true }));
  element.dispatchEvent(new Event('change', { bubbles: true }));
  console.log(`[${new Date().toLocaleTimeString()}] Metnin ilk kısmı yazıldı: "${firstPart}"`);
  await new Promise(resolve => setTimeout(resolve, 1000)); // Listenin güncellenmesi için bekle

  // İkinci kısmı tamamla
  element.value = text;
  element.dispatchEvent(new Event('input', { bubbles: true }));
  element.dispatchEvent(new Event('change', { bubbles: true }));
  console.log(`[${new Date().toLocaleTimeString()}] Metnin tamamı yazıldı: "${text}"`);
}

// Testin tamamlandığını, yükleme (upload) sonucunun ekranda görünmesiyle anlayan fonksiyon.
async function waitForTestCompletion() {
  const uploadResultSelector = '.result-item-upload .result-data-value';
  console.log(`[${new Date().toLocaleTimeString()}] Testin tamamlanması bekleniyor...`);
  const startTime = Date.now();
  const timeout = 60000; // 60 saniye zaman aşımı

  while (Date.now() - startTime < timeout) {
    const uploadResultElement = document.querySelector(uploadResultSelector);
    if (uploadResultElement && uploadResultElement.textContent.trim() !== '' && uploadResultElement.textContent.trim() !== '--') {
      console.log(`[${new Date().toLocaleTimeString()}] Test sonuçları yüklendi, test tamamlandı.`);
      return true;
    }
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  console.error(`[${new Date().toLocaleTimeString()}] Hata: Zaman aşımı. Test sonuçları yüklenemedi.`);
  return false;
}

// Tüm şehirlerdeki sunucular için hız testi döngüsünü başlatan ana fonksiyon.
async function testAllTurkishServers() {
  try {
    const citiesToTest = turkishCities;
    if (citiesToTest.length === 0) {
      console.error("Şehir listesi boş. Lütfen koda şehir adları ekleyin.");
      return;
    }

    for (const city of citiesToTest) {
      console.log(`---`);
      console.log(`[${new Date().toLocaleTimeString()}] Şehir için testler başlatılıyor: ${city}`);
      console.log(`---`);

      // Sunucu seçme ekranını aç
      const changeServerButton = await waitForElement('.btn-server-select');
      if (!changeServerButton) {
        console.error("Sunucu seçme butonu bulunamadı. Lütfen sayfayı yenileyip tekrar deneyin.");
        return;
      }
      changeServerButton.click();
      await new Promise(resolve => setTimeout(resolve, 2000));

      // Şehir arama kutusuna şehir adını yaz
      const searchInput = await waitForElement('#host-search');
      if (!searchInput) {
        console.error("Arama kutusu bulunamadı.");
        const closeButtonIfError = await waitForElement('#find-servers > div > div.pure-u-23-24.u-c > a');
        if (closeButtonIfError) {
          closeButtonIfError.click();
        }
        continue;
      }
      searchInput.value = '';
      searchInput.dispatchEvent(new Event('input'));
      await new Promise(resolve => setTimeout(resolve, 500));

      await typeTextPartially(searchInput, city);
      await new Promise(resolve => setTimeout(resolve, 1000));

      // Arama sonuçlarının yüklenmesini bekle
      const serverNames = await waitForServerListUpdate(city, '.server-hosts-list ul');
      if (serverNames.length === 0) {
        console.warn(`[${new Date().toLocaleTimeString()}] "${city}" için sunucu bulunamadı.`);
        const closeButton = await waitForElement('#find-servers > div > div.pure-u-23-24.u-c > a');
        if (closeButton) {
          closeButton.click();
          console.log(`[${new Date().toLocaleTimeString()}] Arama paneli kapatıldı.`);
        } else {
          console.error("Arama paneli kapatma butonu bulunamadı.");
        }
        continue;
      }

      console.log(`[${new Date().toLocaleTimeString()}] ${city} için bulunan sunucu sayısı: ${serverNames.length}`);

      // Bulunan her sunucu için test yap
      for (const serverName of serverNames) {
        console.log(`---`);
        console.log(`[${new Date().toLocaleTimeString()}] Sunucu için testler başlatılıyor: ${serverName}`);
        console.log(`---`);

        // Sunucu seçim panelini her testten önce tekrar aç
        const changeServerButtonInLoop = await waitForElement('.btn-server-select');
        if (changeServerButtonInLoop) {
          changeServerButtonInLoop.click();
          await new Promise(resolve => setTimeout(resolve, 2000));
        }

        // Arama kutusuna tekrar şehir adını yaz
        const searchInputInLoop = await waitForElement('#host-search');
        if (searchInputInLoop) {
          searchInputInLoop.value = '';
          searchInputInLoop.dispatchEvent(new Event('input'));
          await typeTextPartially(searchInputInLoop, city);
          await new Promise(resolve => setTimeout(resolve, 1000));
        }

        // Sunucu listesinden güncel elementi bul
        const serverListContainer = await waitForElement('.server-hosts-list');
        const serverItems = serverListContainer ? serverListContainer.querySelectorAll('li') : [];
        let serverElementToClick = null;
        for (const item of serverItems) {
          if (item.textContent.trim() === serverName) {
            serverElementToClick = item.querySelector('a');
            break;
          }
        }

        if (serverElementToClick) {
          console.log(`[${new Date().toLocaleTimeString()}] Test ediliyor: ${serverName}`);
          serverElementToClick.click();
          
          // Yeni doğrulama yöntemi: Sunucu isminde şehrin geçip geçmediğini kontrol et
          const serverChanged = await waitForServerChange(serverName, city);
          if (!serverChanged) {
            console.error(`Hata: ${serverName} sunucusu ana sayfada görüntülenemedi. Test atlanıyor.`);
            const closeButton = await waitForElement('#find-servers > div > div.pure-u-23-24.u-c > a');
            if (closeButton) {
              closeButton.click();
            }
            continue;
          }

        } else {
          console.warn(`[${new Date().toLocaleTimeString()}] ${serverName} için seçme butonu bulunamadı. Atlanıyor.`);
          continue;
        }

        // Testi başlat
        const startButton = document.querySelector('.js-start-test');
        if (startButton) {
          startButton.click();
          console.log(`[${new Date().toLocaleTimeString()}] ${serverName} için test başlatıldı.`);

          const testCompleted = await waitForTestCompletion();
          if (testCompleted) {
            console.log(`[${new Date().toLocaleTimeString()}] ${serverName} için test tamamlandı.`);

            const firstStar = await waitForElement('.audience-survey-answers li:first-child a', 60000);
            if (firstStar) {
              firstStar.click();
              console.log(`[${new Date().toLocaleTimeString()}] Anket (tek yıldız) yanıtlandı.`);
              await new Promise(resolve => setTimeout(resolve, 1000));
            } else {
              console.warn("Anket cevabı (tek yıldız) bulunamadı.");
            }
          } else {
            console.error("Test tamamlanamadı veya sonuçlar yüklenmedi.");
          }
        } else {
          console.error("Test başlatma butonu bulunamadı.");
          return;
        }
      }
      
      const closeButtonAtCityEnd = document.querySelector('#find-servers > div > div.pure-u-23-24.u-c > a');
      if (closeButtonAtCityEnd && closeButtonAtCityEnd.offsetParent !== null) {
        closeButtonAtCityEnd.click();
        console.log(`[${new Date().toLocaleTimeString()}] Arama paneli kapatıldı.`);
      }
    }

    console.log("---");
    console.log(`[${new Date().toLocaleTimeString()}] Tüm Türkiye şehirleri için sunucular test edildi. İşlem tamamlandı.`);
    console.log("---");
  } catch (error) {
    console.error(`[${new Date().toLocaleTimeString()}] Betik çalışırken beklenmedik bir hata oluştu:`, error);
  }
}

testAllTurkishServers();

```


Speedtest Automation Script User Guide
This script is designed to run automated speed tests on Speedtest.net for servers in specific cities in Turkey. You don't need to install any software to use the script; a modern web browser is sufficient.

How to Use
Open Speedtest.net: Open your preferred web browser (Chrome, Firefox, Edge, etc.) and navigate to the Speedtest.net website.

Open the Developer Console:

Windows / Linux: Press Ctrl + Shift + J at the same time.

macOS: Press Cmd + Option + J at the same time.

Alternatively, you can find the "Developer Tools" option in the browser menu and click on the "Console" tab.

Paste and Run the Script:

Copy the entire script from the code editor on the right.

In the Console tab that you opened, paste the code into the area where the cursor is blinking.

Press Enter to run the code.

What the Script Does
Once the script is run, it automatically performs the following steps in order for each city in the predefined list (you can change the cities in this list):

Opens the Server Selection Panel: Clicks the "Change Server" button on the main page.

Searches for the City: Types the name of the next city into the search box that appears.

Selects a Server: Selects the first server found and returns to the main page.

Starts the Speed Test: Clicks the speed test start button and waits for the test to complete.

Records the Results: Once the test is complete, it records the results to the console screen.

Answers the Survey: Completes the test by selecting the lowest score (one star) in the survey that appears.

Repeats the Loop: Repeats the same steps for the remaining cities in the list.

Important Notes
Do not interfere with the browser window while the script is running. Otherwise, the script may not be able to find the elements it needs to interact with and may throw an error.

You can edit the turkishCities list at the beginning of the script with the city or location names you want.

If you encounter any problems, error messages will be displayed in the console.

This guide will help you use the script effectively. If you have any questions, please feel free to ask.
