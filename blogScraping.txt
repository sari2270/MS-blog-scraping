const puppeteer = require("puppeteer");
const fs = require("fs");

// helper function: store commenters details
const commentersStoring = (commentersNames, commenters, articlesMapID) => {
  for (const singleCommenter of commentersNames) {
    const name = singleCommenter.name;
    if (!commenters[name]) {
      commenters["" + name] = {
        isMSemployee: singleCommenter.isMSemployee,
        articles: [articlesMapID],
      };
    } else {
      if (!commenters[name]["articles"].includes(articlesMapID)) {
        commenters[name]["articles"].push(articlesMapID);
      }
    }
  }
};

// main function: scraping
(async () => {
  // helper function for recursion
  const commentersByBlogPage = async (url) => {
    // for all 90 pages: go to and generate list of page's urls/articles

    const page = await browser.newPage();
    // page.setDefaultNavigationTimeout(0);
    await Promise.all([page.goto(url), page.waitForNavigation()]);
    // await page.waitForSelector(".card-comments-link");
    let nextPageBtn;
    if (await page.$(".next.page-link")) {
      nextPageBtn = await page.$(".next.page-link");
    }

    // ***for each page of the blog:***
    // find links to articles, only for articles with comments
    const articlesUrlsPerPage = await page.$$eval(
      ".entry-footer .card-comments-link a",
      (links) => links.filter((a) => !(a.innerText == "0")).map((a) => a.href)
    );

    let commenters = {};
    // ***for each article:***
    for (let singleArticleUrl of articlesUrlsPerPage) {
      await page.goto(singleArticleUrl);
      const title = await page.$eval(".entry-title", (h1) => h1.innerText);
      // store article id for the commenter
      articlesMap[articlesMapID] = {
        title,
        url: singleArticleUrl.replace("#comments", ""),
      };
      // find details for the first page of comments
      const commentersNames = await page.$$eval(
        "header .author-name",
        (names) =>
          names.map((span) => {
            const isMSemployee = span.querySelector("a");
            return {
              name: span.innerText.trim(),
              isMSemployee: !!isMSemployee,
            };
          })
      );
      commentersStoring(commentersNames, commenters, articlesMapID);
      // find details for other comments pages, if any
      while (await page.$(".next.page-link")) {
        const button = await page.$(".next.page-link");
        await Promise.all([
          button.evaluate((b) => b.click()),
          page.waitForNavigation(),
        ]);
        // await button.evaluate((b) => b.click());
        // await page.waitForSelector(".entry-title");
        const commentersNames = await page.$$eval(
          "header .author-name",
          (names) =>
            names.map((span) => {
              const isMSemployee = span.querySelector("a");
              return {
                name: span.innerText.trim(),
                isMSemployee: !!isMSemployee,
              };
            })
        );
        commentersStoring(commentersNames, commenters, articlesMapID);
      }
      articlesMapID++;
    }
    await page.close();

    if (!nextPageBtn) return commenters;
    else {
      const nextPageNumber = parseInt(url.match(/(\d+)$/)[1], 10) + 1;
      const nextUrl = `https://devblogs.microsoft.com/visualstudio/page/${nextPageNumber}`;
      const nextPageCommenters = await commentersByBlogPage(nextUrl);
      for (const k in nextPageCommenters) {
        if (!commenters[k]) {
          commenters[k] = nextPageCommenters[k];
        } else {
          commenters[k].articles.push(...nextPageCommenters[k].articles);
        }
      }
      return commenters;
    }
  };

  const browser = await puppeteer.launch({ headless: false });
  const firstUrl = "https://devblogs.microsoft.com/visualstudio/page/1";
  const articlesMap = {};
  let articlesMapID = 0;
  const commenters = await commentersByBlogPage(firstUrl);
  // convert the commenters object to an array
  const finalCommenters = Object.keys(commenters).map((commenter) => {
    // replace the article ID with the article object
    const articlesPerCommenter = commenters[commenter].articles.map(
      (article) => articlesMap[article]
    );
    return {
      name: commenter,
      isMSemployee: commenters[commenter].isMSemployee,
      articles: articlesPerCommenter,
    };
  });
  fs.writeFileSync(
    "commenters.json",
    JSON.stringify(finalCommenters, null, 2),
    "utf-8"
  );
  console.log(`
**************************************************************************
${finalCommenters.length} commenters were found. Go to ./commenters.json to get their details
**************************************************************************
    `);

  await browser.close();
})();
