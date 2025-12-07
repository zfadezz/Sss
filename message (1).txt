// install first:
// npm install puppeteer
// (Node 18+ recommended so `fetch` is built-in. If not, install node-fetch.)

import puppeteer from "puppeteer";

// 🔐 Prefer env var: OPENROUTER_API_KEY="sk-or-..."
const OPENROUTER_API_KEY = process.env.OPENROUTER_API_KEY || "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";

// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY
// FADEZZZ GET A NEW KEY THIS ONE IS EXPIRED: https://openrouter.com/ MAKE AN ACCOUNT ON THAT WEBSITE AND GET YOUR KEY

// Simple delay helper
const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function requestGenerationFromNode(prompt, extract) {
    if (!OPENROUTER_API_KEY || OPENROUTER_API_KEY === "YOUR_OPENROUTER_API_KEY_HERE") {
        throw new Error("OPENROUTER_API_KEY is not set. Use an env var or update the constant.");
    }

    const systemPrompt = `
You will answer the question at the very bottom of the message that is written under Q1-Q4. 
Each new line under the question is a new possible answer. 
Do not be afraid to answer "Not in the Story".
The extract will be provided first, then the questions.
You must provide ONLY the chosen answer from the possible answers, word for word, and nothing else.
  `.trim();

    const body = {
        model: "meta-llama/llama-3.3-70b-instruct:free",
        messages: [
            { role: "system", content: systemPrompt },
            { role: "user", content: extract },
            {
                role: "assistant",
                content: "I have acknowledged the extract and will use it to answer your questions.",
            },
            { role: "user", content: prompt },
        ],
    };

    const res = await fetch("https://openrouter.ai/api/v1/chat/completions", {
        method: "POST",
        headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${OPENROUTER_API_KEY}`,
            "HTTP-Referer": "https://your-tool-or-local-script",
            "X-Title": "Sparx Reader Helper",
        },
        body: JSON.stringify(body),
    });

    if (!res.ok) {
        const text = await res.text().catch(() => "");
        throw new Error("OpenRouter request failed: " + text);
    }

    const data = await res.json();
    const answer = data?.choices?.[0]?.message?.content?.trim?.() ?? "Not in the Story";

    return answer;
}

// ---- Helpers that run in Node and use page.evaluate inside ----

async function goToLibrary(page) {
    // Wait for the Library button to be present and click it
    await page.waitForSelector('div[role="button"] span', { timeout: 10000 });

    await page.evaluate(() => {
        const btn = Array.from(document.querySelectorAll('div[role="button"]'))
            .find((el) => {
                const span = el.querySelector("span");
                return span && span.textContent.trim().toLowerCase() === "library";
            });

        btn?.click();
    });
}

async function getAllBooks(page) {
    return await page.evaluate(() => {
        const bookElements = Array.from(
            document.querySelectorAll('[data-test-id^="library-book-"] img[alt]')
        );

        return bookElements.map((img) => {
            let title = img.alt || "";
            title = title.replace(/book cover/i, "").trim();

            return {
                title,
                src: img.src,
                id: img.closest("[data-test-id]")?.getAttribute("data-test-id") || null,
            };
        });
    });
}

async function findBookByTitle(page, searchTitle) {
    const books = await getAllBooks(page);

    const match = books.find(
        (b) => b.title.toLowerCase() === searchTitle.toLowerCase()
    );

    return match || null;
}

async function removeGoldOrCompletedSection(page) {
    await page.evaluate(() => {
        const findHeading = (text) =>
            Array.from(document.querySelectorAll("h2")).find(
                (h) => h.textContent.trim().toLowerCase() === text.toLowerCase()
            );

        // First remove Gold Reader books section if present
        let heading = findHeading("Gold Reader books");
        if (heading) {
            let el = heading.nextElementSibling;
            while (el && el.tagName.toLowerCase() === "div") {
                const next = el.nextElementSibling;
                el.remove();
                el = next;
            }
        }

        // Then remove Completed books section if present
        heading = findHeading("Completed books");
        if (heading) {
            let el2 = heading.nextElementSibling;
            while (el2 && el2.tagName.toLowerCase() === "div") {
                const next = el2.nextElementSibling;
                el2.remove();
                el2 = next;
            }
        }
    });
}

async function clickBookByTitle(page, title) {
    const found = await page.evaluate((searchTitle) => {
        const books = Array.from(
            document.querySelectorAll('[data-test-id^="library-book-"]')
        );

        for (const book of books) {
            const img = book.querySelector("img[alt]");
            if (!img) continue;

            let name = img.alt.replace(/book cover/i, "").trim();

            if (name.toLowerCase() === searchTitle.toLowerCase()) {
                book.scrollIntoView({ behavior: "smooth", block: "center" });
                book.click();
                return true;
            }
        }

        return false;
    }, title);

    if (!found) {
        throw new Error(`Book "${title}" not found`);
    }
}

async function clickStartReading(page) {
    await page.waitForSelector("button", { timeout: 15000 });

    await page.evaluate(() => {
        // Find all buttons of type="button"
        const buttons = document.querySelectorAll('button[type="button"]');

        // Loop through them to match based on content
        for (let button of buttons) {
            // Check if it contains a div with exact text "Continue Reading"
            const textDiv = button.querySelector('div');
            if (textDiv && textDiv.textContent.trim() === 'Continue Reading') {
                // Also confirm the presence of the chevron-right SVG
                const svg = button.querySelector('svg[data-icon="chevron-right"]');
                if (svg) {
                    // Click the button if both match
                    button.click();
                    console.log('Button clicked successfully.');
                    break; // Stop after clicking the first match
                }
            }
        }
    });
}

// ---- Main ----

async function start(loginType, username, password, schoolName) {
    try {
        const browser = await puppeteer.launch({
            headless: false,
            defaultViewport: null,
        });

        const page = await browser.newPage();

        // Pipe page console logs to Node console
        page.on("console", (msg) => {
            console.log("PAGE:", msg.text());
        });

        // (Optional) handle dialogs if you later use alert() in page
        page.on("dialog", async (dialog) => {
            console.log("DIALOG:", dialog.message());
            await dialog.dismiss();
        });

        // Go to reader
        await page.goto("https://reader.sparx-learning.com", {
            waitUntil: "networkidle2",
        });

        console.log("Page opened!");

        if (loginType === "microsoft") {
            // Microsoft login flow
            await page.goto("https://auth.sparx-learning.com/", {
                waitUntil: "networkidle2",
            });

            const schoolInputSelector = 'input[class*="_Input_"]';
            await page.waitForSelector(schoolInputSelector);
            await page.type(schoolInputSelector, schoolName);

            await page.waitForFunction(
                () =>
                    !!document.querySelector(
                        "._SchoolResult_1h7n6_1._Selected_1h7n6_10"
                    )
            );

            await page.evaluate(() =>
                document
                    .querySelector("._SchoolResult_1h7n6_1._Selected_1h7n6_10")
                    ?.click()
            );

            await page.evaluate(() =>
                Array.from(document.querySelectorAll("button")).find(
                    (b) => b.textContent.trim() === "Continue"
                )?.click()
            );

            await page.waitForNavigation({ waitUntil: "networkidle2" });

            try {
                await page.waitForSelector("#cookiescript_accept", { timeout: 3000 });
                await page.click("#cookiescript_accept");
            } catch { }

            await page.evaluate(() => {
                document.getElementById("cookiescript_accept")?.click();
            });

            await page.evaluate(() =>
                document
                    .querySelector(".sso-options-container .sso-login-button")
                    ?.click()
            );

            await page.waitForNavigation({ waitUntil: "networkidle2" });

            await page.waitForSelector("#i0116");
            await page.type("#i0116", username);
            await page.click("#idSIButton9");

            await page.waitForNavigation({ waitUntil: "networkidle2" });

            await delay(5000);

            await page.waitForSelector("#i0118");
            await page.type("#i0118", password);
            await delay(5000);

            await page.click("#idSIButton9");
            await page.waitForNavigation({ waitUntil: "networkidle2" });

            try {
                await page.waitForSelector("#idSIButton9:not([disabled])", {
                    timeout: 4000,
                });
                await page.click("#idSIButton9");
            } catch { }

            await page.waitForSelector(
                'a[href="https://reader.sparx-learning.com"]'
            );

            await page.click('a[href="https://reader.sparx-learning.com"]');
        }

        await delay(5000);

        // ---- SRP CHECKER ----

        async function getSRP(page) {
            return await page.evaluate(() => {
                const srpSpan = Array.from(document.querySelectorAll("span"))
                    .find(el => el.textContent.trim() === "SRP");

                if (!srpSpan) return null;

                const parent = srpSpan.parentElement;
                if (!parent) return null;

                const text = parent.innerText.replace(/\s*SRP\s*/i, "").trim();
                return Number(text.replace(/,/g, ""));
            });
        }

        // record initial SRP
        const startingSRP = await getSRP(page);
        console.log("Starting SRP:", startingSRP);

        if (startingSRP === null) {
            console.log("Failed to detect SRP — skipping monitoring.");
        } else {
            // monitor every second
            const checker = setInterval(async () => {
                try {
                    const currentSRP = await getSRP(page);

                    if (currentSRP === null) return;

                    const gained = currentSRP - startingSRP;

                    console.log("Current SRP:", currentSRP, "Gained:", gained);

                    if (gained >= 300) {
                        console.log("🔥 SRP gain exceeded 300! Stopping...");
                        clearInterval(checker);

                        // Optional: stop browser
                        // await browser.close();

                    }
                } catch (err) {
                    console.error("SRP checker error:", err);
                }
            }, 1000);
        }

        // Go to Library
        await goToLibrary(page);

        // Clean up Gold/Completed sections
        await removeGoldOrCompletedSection(page);

        // Show books in console (you can send this to Discord)
        const books = await getAllBooks(page);
        console.log("Books found:", books);

        const targetTitle = "Sherlock Holmes: The Adventure of the Beryl Coronet";

        const sherlock = await findBookByTitle(page, targetTitle);
        console.log("Match:", sherlock);

        // Click that book
        await clickBookByTitle(page, targetTitle);
        console.log("Book clicked!");

        await delay(5000);

        // Click "Start Reading"
        await clickStartReading(page);
        console.log("Started reading!");


        async function clickReadUpToHere(page) {
            await page.waitForFunction(() => {
                const btn = document.querySelector('button[data-test-id="read-button"]');
                if (!btn) return false;

                const text = btn.textContent.trim().toLowerCase();
                return text.includes("i have read up to here");
            }, { timeout: 15000 });

            await page.evaluate(() => {
                const btn = document.querySelector('button[data-test-id="read-button"]');
                if (btn) {
                    btn.scrollIntoView({ behavior: "smooth", block: "center" });
                    btn.click();
                    console.log("Clicked: I have read up to here");
                }
            });
        }

        await page.exposeFunction("clickAnswerButton", async (answerText) => {
            await page.evaluate((text) => {
                // Normalize whitespace and lowercase for safe comparison
                const normalize = (s) =>
                    s.trim().replace(/\s+/g, " ").toLowerCase();

                const buttons = Array.from(document.querySelectorAll("button"));

                const target = buttons.find(btn =>
                    normalize(btn.textContent) === normalize(text)
                );

                if (target) {
                    target.scrollIntoView({ behavior: "smooth", block: "center" });
                    target.click();
                    console.log("Clicked answer:", text);
                } else {
                    console.log("Button not found for:", text);
                }
            }, answerText);
        });


        // ---- AI integration part ----

        // Expose Node-side function to page
        await page.exposeFunction("requestGenerationFromNode", requestGenerationFromNode);

        await page.evaluate(async () => {
            const sleep = (ms) => new Promise(res => setTimeout(res, ms));

            console.log("Waiting inside page…");
            await sleep(1500); // 1.5 seconds
            console.log("Done inside page!");
        });

        async function clickNextWithDelay(page, delayMs = 1000) {
            // 1) Wait until the correct Next button actually exists
            await page.waitForFunction(() => {
                const buttons = document.querySelectorAll('button[type="button"]');
                for (let button of buttons) {
                    if (button.getAttribute('data-test-id') === 'next-continue-button') {
                        const textDiv = button.querySelector('div');
                        const svg = button.querySelector('svg[data-icon="chevron-right"]');
                        if (textDiv && textDiv.textContent.trim() === 'Next' && svg) {
                            return true;
                        }
                    }
                }
                return false;
            }, { timeout: 15000 });

            // 2) Once it's there, run your click logic inside the page with a delay
            await page.evaluate(async (delayMs) => {
                const sleep = (ms) => new Promise(res => setTimeout(res, ms));

                const buttons = document.querySelectorAll('button[type="button"]');

                for (let button of buttons) {
                    if (button.getAttribute('data-test-id') === 'next-continue-button') {
                        const textDiv = button.querySelector('div');
                        const svg = button.querySelector('svg[data-icon="chevron-right"]');

                        if (textDiv && textDiv.textContent.trim() === 'Next' && svg) {
                            console.log(`Next button found, waiting ${delayMs}ms before click...`);
                            await sleep(delayMs);
                            button.click();
                            console.log('Next button clicked after delay.');
                            break;
                        }
                    }
                }
            }, delayMs);
        }

        // Inject observer
        await page.evaluate(() => {
            // 1. Give the interval a name so we can stop it later


                continueClicker = setInterval(() => {
                    // Find ALL buttons and then find the one whose innerText is exactly "Continue" after trimming.
                    const button = [...document.querySelectorAll('button')].find(
                        // We check for innerText existence first to prevent errors on empty elements
                        b => b.innerText && b.innerText.trim() === "Continue"
                    );

                    if (button) {
                        // Click the button if it's found
                        button.click();
                        console.log("✅ 'Continue' button clicked!");

                    } else {
                        console.log("⚠️ 'Continue' button not found (waiting...)");
                    }
                }, 1000); // Click every 1000 milliseconds (1 second)
            

            // 1. Give the interval a name so we can stop it later
            let continueReadingClicker = null;


            // 2. Define the click function
            const startContinueClicker = () => {
                continueReadingClicker = setInterval(() => {

                    // Find ALL buttons and then use .find() to locate the one with the correct text
                    const button = [...document.querySelectorAll('button')].find(
                        b => b.innerText && b.innerText.trim() === "Continue reading"
                    );

                    if (button) {
                        // Click the button if it's found
                        button.click();
                        console.log("✅ 'Continue reading' button clicked!");

                        // Optional: Stop the script once the button is successfully clicked
                        // clearInterval(continueReadingClicker);
                        // console.log("🛑 Found and clicked the button, auto-clicker stopped.");

                    } else {
                        console.log("⚠️ 'Continue reading' button not found (waiting...)");
                    }

                }, 1000); // Click every 1000 milliseconds (1 second)
            };

            // 3. Start the script
            startContinueClicker();
            // 1. Create a unique ID for the interval so we can stop it later
            const autoClicker = setInterval(() => {


                // 2. Find the button using the stable data-test-id
                const button = document.querySelector('[data-test-id="next-continue-button"]');

                // 3. Check if button exists and isn't disabled
                if (button) {
                    button.click();
                    console.log("✅ Button clicked");
                } else {
                    // console.log("⚠️ Button not found (waiting...)");
                }

            }, 1000); // 1000 milliseconds = 1 second

            // 1. Give the interval a name so we can stop it
            let retryClicker = null;

            // 2. Define the click function
            const startRetryClicker = () => {
                retryClicker = setInterval(() => {

                    // XPath to find a button element that contains the exact text "Retry"
                    const xPath = "//button[.//text()='Retry']";

                    // Evaluate the XPath expression
                    const result = document.evaluate(xPath, document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null);
                    const button = result.singleNodeValue;

                    if (button) {
                        button.click();
                        console.log("✅ 'Retry' button clicked!");
                    } else {
                        // console.log("⚠️ 'Retry' button not found (waiting...)");
                        // Optional: If the button is gone for a while, you might want to stop the script:
                        // clearInterval(retryClicker); 
                        // console.log("🛑 Retry button disappeared. Auto-clicker stopped.");
                    }

                }, 1000); // Click every 1000 milliseconds (1 second)
            };

            // 3. Start the script
            startRetryClicker();

            console.log("The script is currently working inside the page!");

            let lastExtract = "";
            let lastQuestion = "";
            let startingValue = "Q0.";

            function copyTextParagraph(startAt, endAt) {
                const allText = document.body.innerText;

                const startIndex = allText.indexOf(startAt);
                if (startIndex === -1) return "";

                if (endAt) {
                    const afterStartIndex = startIndex + startAt.length;
                    const endIndex = allText.indexOf(endAt, afterStartIndex);
                    if (endIndex === -1) {
                        return allText.substring(afterStartIndex).trim();
                    }
                    return allText.substring(afterStartIndex, endIndex).trim();
                }

                return allText.substring(startIndex).trim();
            }

            const observer = new MutationObserver(async () => {
                const extractRead = document.querySelector('[class^="read-content"]');
                const questionRead = document.querySelector(
                    '[class="PanelPaperbackQuestionContainer"]'
                );

                if (extractRead) {
                    const copiedText = copyTextParagraph(
                        "Start reading here",
                        "Stop reading here"
                    );

                    if (lastExtract !== copiedText && copiedText) {

                        const wordCount = copiedText.split(" ").length;

                        console.log(wordCount); // 2

                        // 1. A helper function to create a pause (returns a Promise)
                        const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms));

                        // 2. The robust word counter from before
                        function countWords(str) {
                            const trimmed = str.trim();
                            return trimmed === "" ? 0 : trimmed.split(/\s+/).length;
                        }

                        // 3. The main function with the dynamic delay
                        async function displayMessageWithDelay(message) {
                            const wordCount = countWords(message);

                            // Calculate delay: 1 word = 1000 milliseconds (1 second)
                            const delayTime = wordCount * 100;

                            console.log(`Analyzing: "${message}"`);
                            console.log(`Word count: ${wordCount}`);
                            console.log(`Waiting for ${delayTime / 1000} seconds...`);

                            // This line pauses the code execution!
                            await wait(delayTime);

                            // This runs only AFTER the delay finishes
                            console.log("✅ Message Displayed!");
                        }

                        // Example Usage:
                        // This has 3 words, so it will wait 3 seconds.
                        await displayMessageWithDelay(copiedText);

                        const btn = document.querySelector('button[data-test-id="read-button"]');
                        if (btn) {
                            btn.scrollIntoView({ behavior: "smooth", block: "center" });
                            btn.click();
                            console.log('Clicked "I have read up to here"');
                        }

                        lastExtract = copiedText;
                        console.log("Extract:", copiedText);
                    }
                }

                if (questionRead) {
                    let copiedText = copyTextParagraph("Q");

                    const matches = [...copiedText.matchAll(/Q\d+\./g)];
                    let updatedText = copiedText;

                    if (matches.length > 1) {
                        updatedText = copiedText.replace(
                            /Q\d+\.[\s\S]*?(?=Q\d+\.)/,
                            ""
                        );
                    }

                    copiedText = updatedText.trim();

                    if (
                        copiedText &&
                        copiedText !== lastQuestion &&
                        !copiedText.startsWith(startingValue)
                    ) {
                        startingValue = copiedText.slice(0, 3);
                        lastQuestion = copiedText;
                        console.log("Question block:", copiedText);

                        if (!lastExtract) {
                            console.log("No extract to copy!");
                            return;
                        }

                        // Call Node-side OpenRouter function
                        window
                            .requestGenerationFromNode(copiedText, lastExtract)
                            .then(async (result) => {
                                console.log("Generated text:", result);

                                // now click the answer
                                window.clickAnswerButton(result);


                                await clickNextWithDelay(page, 2000);
                            })
                            .catch((err) => {
                                console.error("OpenRouter error:", err);
                            });

                    }
                }
            });

            observer.observe(document.body, {
                childList: true,
                subtree: true,
                characterData: true,
            });
        });

        console.log("Observer attached. Wait for questions and watch the logs.");

        // Keep browser open
        // await browser.close();
    } catch (error) {
        console.error("An error occurred:", error);
    }
}

start(
    "microsoft",
    "user@email.com",
    "********",
    "ur school here"
);
