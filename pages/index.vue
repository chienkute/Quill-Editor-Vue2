<template>
  <div class="quill-container">
    <quill-editor
      class="editor"
      ref="myTextEditor"
      :value="content"
      :options="editorOption"
      @change="onEditorChange"
      @blur="onEditorBlur($event)"
      @focus="onEditorFocus($event)"
      @ready="onEditorReady($event)"
    />
    <!-- <div class="output ql-snow">
      <div class="title">Output</div>
      <div class="ql-editor" v-html="content"></div>
    </div> -->
    <!-- <div class="emoji">
      <div @click="toogleDialogEmoji">😃</div>
      <div v-if="emojiShow">
        <Picker
          :data="emojiIndex"
          set="google"
          @select="showEmoji"
          :autoFocus="true"
          :showSkinTones="false"
          :showPreview="false"
          title="Sharaku"
          :i18n="{
            search: '検索',
            notfound: '絵文字が見つかりません',
            categories: {
              search: '検索結果',
              recent: 'よく使うもの',
              smileys: 'スマイリー＆顔文字',
              people: '人物＆体',
              nature: '動物＆自然',
              foods: '食べ物＆飲み物',
              activity: '活動',
              places: '旅行＆場所',
              objects: '物',
              symbols: '記号',
              flags: '旗',
              custom: 'カスタム',
            },
          }"
        />
      </div>

      <div>
        <div>
          {{ emojisOutput }}
        </div>
      </div>
    </div> -->
    <button class="button" @click="exportToJson">Export to JSON</button>
    <div v-html="htmlData"></div>
    <div class="data">{{ formattedDataHtml }}</div>
  </div>
</template>

<script>
import { quillEditor } from "vue-quill-editor";
import "quill-emoji/dist/quill-emoji.css";
import quillEmoji from "quill-emoji";
import mention from "quill-mention";
import dedent from "dedent";
import "quill/dist/quill.core.css";
import "quill/dist/quill.snow.css";
import "quill/dist/quill.bubble.css";
import "quill-mention/dist/quill.mention.css";
import data from "emoji-mart-vue-fast/data/all.json";
import "emoji-mart-vue-fast/css/emoji-mart.css";
import { Picker, EmojiIndex } from "emoji-mart-vue-fast";
import { parseDocument, DomUtils } from "htmlparser2";

let emojiIndex = new EmojiIndex(data);
export default {
  name: "IndexPage",
  components: {
    quillEditor,
    Picker,
  },
  data() {
    return {
      editorOption: {
        placeholder: "テキストをここに入力してください",
        modules: {
          "emoji-toolbar": true,
          "emoji-shortname": true,
          mention: {
            allowedChars: /^[A-Za-z\s]*$/,
            mentionDenotationChars: ["@", "#"],
            source: (searchTerm, renderList) => {
              const values = [
                { id: 1, value: "Alice" },
                { id: 2, value: "Bob" },
                { id: 3, value: "Charlie" },
                { id: 4, value: "David" },
              ];

              if (searchTerm.length === 0) {
                renderList(values, searchTerm);
              } else {
                const matches = values.filter((item) =>
                  item.value.toLowerCase().includes(searchTerm.toLowerCase())
                );
                renderList(matches, searchTerm);
              }
            },
          },
          toolbar: {
            container: [
              ["bold", "italic", "strike"],
              [{ list: "ordered" }, { list: "bullet" }],
              ["link", "code-block", "blockquote", "code"], // Links, Code blocks
              [{ align: [] }],
              ["image", "emoji"],
            ],
          },
        },
      },
      content: "",
      emojiIndex: emojiIndex,
      emojisOutput: "",
      emojiShow: false,
      formattedDataHtml: [],
      html: "",
      nodes: [
        "ul",
        "ol",
        "li",
        "pre",
        "blockquote",
        "img",
        "h1",
        "h2",
        "h3",
        "h4",
        "h5",
        "h6",
        "p",
      ],
    };
  },
  methods: {
    onEditorChange(value) {
      this.content = value.html;
    },
    exportToJson() {
      this.html = this.editor.root.innerHTML;
      this.formattedDataHtml = this.parseHtml(this.html);
      const dataConverToHtml = this.convertBlocksToHTML(
        this.formattedDataHtml[0].elements
      );
      this.htmlData = dataConverToHtml;
      console.log("file html ne ", this.formattedDataHtml);
      console.log("file html ne ", dataConverToHtml);
    },
    onEditorBlur(editor) {
      console.log("editor blur!", editor);
    },
    onEditorFocus(editor) {
      console.log("editor focus!", editor);
    },
    onEditorReady(editor) {
      console.log("editor ready!", editor);
    },
    showEmoji(emoji) {
      console.log("show emoji!", emoji);
      this.emojisOutput = this.emojisOutput + emoji.native;
      let lastIndex = this.editor.getText().length - 1;
      this.editor.insertText(lastIndex, emoji.native);
    },
    toogleDialogEmoji() {
      this.emojiShow = !this.emojiShow;
    },
    inearizeLists(input) {
      const document = parseDocument(input);
      const result = [];
      function processList(listElement, indentLevel) {
        const listType = listElement.tagName;
        const listItems = [];
        DomUtils.getChildren(listElement).forEach((child) => {
          if (DomUtils.isTag(child) && child.tagName === "li") {
            const listItemContent = DomUtils.getInnerHTML(child)
              .trim()
              .replace(/<ul[\s\S]*?<\/ul>/g, "")
              .replace(/<ol[\s\S]*?<\/ol>/g, "")
              .replace(/<li[\s\S]*?<\/li>/g, "");
            listItems.push(`<li>${listItemContent}</li>`);
            const nestedList = DomUtils.getChildren(child).find(
              (nestedChild) =>
                DomUtils.isTag(nestedChild) &&
                (nestedChild.tagName === "ul" || nestedChild.tagName === "ol")
            );
            if (nestedList) {
              result.push(
                `<${listType} indent=${indentLevel}>${listItems.join(
                  ""
                )}</${listType}>`
              );
              listItems.length = 0;
              processList(nestedList, indentLevel + 1);
            }
          }
        });
        if (listItems.length > 0) {
          result.push(
            `<${listType} indent=${indentLevel}>${listItems.join(
              ""
            )}</${listType}>`
          );
        }
      }
      DomUtils.getElementsByTagName("ul", document)
        .concat(DomUtils.getElementsByTagName("ol", document))
        .forEach((element) => {
          const parent = DomUtils.getParent(element);
          if (
            parent &&
            DomUtils.isTag(parent) &&
            (parent.tagName === "li" ||
              parent.tagName === "ul" ||
              parent.tagName === "ol")
          )
            return;
          processList(element, 0);
          const newElement = result.join("");
          input = input.replace(DomUtils.getOuterHTML(element), newElement);
          result.length = 0;
        });
      return input;
    },
    blockBuilder(array) {
      const blocks = [];
      let richTextBlocks = {
        type: "rich_text",
        elements: [],
      };
      for (let i = 0; i < array.length; i++) {
        const block = array[i];
        if (block.type === "image" || block.type === "header") {
          if (richTextBlocks.elements.length > 0) {
            blocks.push(richTextBlocks);
            richTextBlocks = {
              type: "rich_text",
              elements: [],
            };
          }
          blocks.push(block);
        } else {
          richTextBlocks.elements.push(block);
        }
      }
      if (richTextBlocks.elements.length > 0) {
        blocks.push(richTextBlocks);
      }
      return blocks;
    },
    findDomElementsByTagName(dom, tag) {
      return DomUtils.findAll(
        (node) => node.type === "tag" && node.name === tag,
        dom
      );
    },
    parseNodeElement(node) {
      const block = {};
      switch (node.type) {
        case "tag": {
          switch (node.name) {
            case "b":
            case "u":
            case "strong":
            case "i":
            case "em":
            case "s":
            case "del":
            case "code":
            case "a":
              return this.parseText(node);
            default:
              break;
          }
          break;
        }
        case "text":
          return this.parseText(node);
      }
      return block;
    },
    parseNode(node) {
      let block = {};
      switch (node.type) {
        case "tag": {
          switch (node.name) {
            case "ul":
            case "ol":
              block = {
                type: "rich_text_list",
                style: node.name === "ul" ? "bullet" : "ordered",
                elements: [],
                indent: parseInt(node.attribs["indent"]),
              };
              break;
            case "li":
              block = {
                type: "rich_text_section",
                elements: [],
              };
              break;
            case "pre":
              block = {
                type: "rich_text_preformatted",
                elements: [],
              };
              break;
            case "blockquote":
              block = {
                type: "rich_text_quote",
                elements: [],
              };
              break;
            case "img":
              block = {
                type: "image",
                image_url: node.attribs.src,
                alt_text: node.attribs.alt || "",
              };
              if (node.attribs.title) {
                block.title = {
                  type: "plain_text",
                  text: node.attribs.title,
                };
              }
              break;
            case "p":
              block = {
                type: "rich_text_section",
                elements: [],
              };
              break;
            case "h1":
            case "h2":
            case "h3":
            case "h4":
            case "h5":
            case "h6":
              return this.parseHeader(node);
          }
          if (block && node.children) {
            for (const child of node.children) {
              const childObj =
                child.type === "tag" && this.nodes.includes(child.name)
                  ? this.parseNode(child)
                  : this.parseNodeElement(child);
              if (childObj && "elements" in block) {
                if (!Array.isArray(childObj)) {
                  block.elements.push(...[childObj]);
                } else {
                  block.elements.push(...childObj); // Flatten the array
                }
                block.elements = block.elements.filter(
                  (element) => Object.keys(element).length !== 0
                );
              }
            }
          }
          break;
        }
      }
      return block;
    },
    parseHeader(node) {
      const header = {};
      if (
        node.children &&
        node.children[0] &&
        node.children[0].type === "text" &&
        node.children[0].data
      ) {
        header.type = "header";
        header.text = {
          type: "plain_text",
          text: node.children[0].data,
        };
      }
      return header;
    },
    parseText(node, style = {}) {
      const texts = [];
      if (node.type === "tag") {
        switch (node.name) {
          case "b":
          case "strong":
            style.bold = true;
            break;
          case "i":
          case "em":
            style.italic = true;
            break;
          case "s":
          case "del":
            style.strike = true;
            break;
          case "code":
            style.code = true;
            break;
          default:
            break;
        }
        if (node.name === "a") {
          const textElement = {
            type: "link",
            url: node.attribs.href,
          };
          if (Object.keys(style).length > 0) {
            textElement.style = Object.assign({}, style);
          }
          if (node.children) {
            if (node.children[0].type === "text") {
              textElement.text = node.children[0].data;
            }
          }
          texts.push(textElement);
        } else {
          for (const child of node.children) {
            texts.push(...this.parseText(child, Object.assign({}, style)));
          }
        }
      } else if (node.type === "text") {
        const textElement = {
          type: "text",
          text: node.data.replace(/<br ?\/?>/g, "\n").replace(/^\s*$/, ""),
        };
        if (Object.keys(style).length > 0) {
          textElement.style = Object.assign({}, style);
        }
        texts.push(textElement);
      }
      return texts;
    },
    compressHTML(html) {
      // Remove all malformed tags
      const document = parseDocument(html);
      const correctedHtml = DomUtils.getOuterHTML(document);
      html = correctedHtml;
      // Use a regular expression to find content within <pre> tags and store it. SPACES
      const preTags = [];
      const preLeadingSpaces = [];
      html = html.replace(
        /<p>(.*?)<img([^>]*)\/?>(.*?)<\/p>/g,
        (match, beforeImage, imgAttributes, afterImage) => {
          // Format the separated parts as:
          // <p>before the image</p><img ... /><p>after the image</p>
          let newHtml = "";
          // Handle the text before the image
          newHtml += `<p>${beforeImage.trim()}</p>`;
          // Reconstruct the img tag
          newHtml += `<img${imgAttributes}/>`;
          // Handle the text after the image
          newHtml += `<p>${afterImage.trim()}</p>`;
          return newHtml;
        }
      );
      html = html.replace(/<p>(\s*(<br\s*\/?>)\s*)*<\/p>/g, "");
      html = html.replace(
        /([ \t]*)<pre>([\s\S]*?)<\/pre>/g,
        (match, leadingSpaces, content) => {
          preTags.push(content.replace(/^\n/, ""));
          preLeadingSpaces.push(leadingSpaces);
          return `<pre>${preTags.length - 1}</pre>`;
        }
      );
      // Use a regular expression to find tag attributes and store them.
      const tagAttributes = [];
      html = html.replace(/<(\w+)([^>]*)>/g, (match, tagName, attributes) => {
        tagAttributes.push(attributes);
        return `<${tagName} data-tag-attr="${tagAttributes.length - 1}">`;
      });
      // Remove all newlines and extra spaces from the HTML string
      html = html.replace(/\n[\n ]*/g, "");
      // Restore the original tag attributes
      html = html.replace(
        /<(\w+) data-tag-attr="(\d+)">/g,
        (match, tagName, index) => {
          return `<${tagName}${tagAttributes[parseInt(index)]}>`;
        }
      );
      // Restore the original content within <pre> tags
      html = html.replace(/<pre>(\d+)<\/pre>/g, (match, index) => {
        const content = preTags[parseInt(index)];
        const leadingSpaces = preLeadingSpaces[parseInt(index)];
        // Remove the leading spaces from each line in the content
        const adjustedContent = content.replace(
          new RegExp("^" + leadingSpaces, "mg"),
          ""
        );
        return `<pre>${adjustedContent}</pre>`;
      });
      html = html
        .replace(/<br ?\/?>/g, "\n")
        .replace(/<\/?span[^>]*>/g, "")
        .replace(/<\/?div[^>]*>/g, "");
      // Add spaces around inline elements within <p> tags
      html = html
        .replace(/<p>([\s\S]*?)<\/p>/g, (match, content) => {
          // Ensure there are spaces around inline tags
          content = content.replace(/(\S)(<b>|<i>|<code>)/g, "$1 $2"); // space before tags
          return `<p>${content}\n</p>`;
        })
        .replace(/\n+<\/p>/g, "\n</p>")
        .replace(/>\n<\/p>/g, "></p>");
      return html;
    },
    parseHtml(html) {
      html = this.compressHTML(html);
      html = this.inearizeLists(html);
      const dom = parseDocument(html);
      const body =
        this.findDomElementsByTagName(dom.children, "body")[0] || dom;
      const blocks = [];
      body.children.forEach((node) => {
        blocks.push(this.parseNode(node));
      });
      const blocksObj = this.blockBuilder(blocks);
      return blocksObj;
    },
    convertBlocksToHTML(blocks) {
      let html = "";

      blocks.forEach((block) => {
        switch (block.type) {
          case "rich_text_section":
            html += this.convertTextSection(block);
            break;
          case "rich_text_list":
            html += this.convertList(block);
            break;
          case "rich_text_preformatted":
            html += this.convertPreformatted(block);
            break;
          case "rich_text_quote":
            html += this.convertQuote(block);
            break;
          case "image":
            html += this.convertImage(block);
            break;
          case "header":
            html += this.convertHeader(block);
            break;
          default:
            break;
        }
      });

      return html;
    },

    convertTextSection(block) {
      let html = "";
      block.elements.forEach((element) => {
        if (element.type === "text") {
          let text = element.text;
          if (element.style) {
            // Apply styles like bold, italic, etc.
            if (element.style.bold) text = `<b>${text}</b>`;
            if (element.style.italic) text = `<i>${text}</i>`;
            if (element.style.strike) text = `<s>${text}</s>`;
            if (element.style.code) text = `<code>${text}</code>`;
          }
          html += text;
        } else if (element.type === "link") {
          let link = `<a href="${element.url}"`;
          if (element.style) {
            // Add inline styles if needed
            link += this.convertStyles(element.style);
          }
          link += `>${element.text}</a>`;
          html += link;
        }
      });
      return `<p>${html}</p>`;
    },

    convertList(block) {
      let listHtml = block.style === "bullet" ? "<ul>" : "<ol>";
      block.elements.forEach((item) => {
        listHtml += `<li>${this.convertTextSection(item)}</li>`;
      });
      listHtml += block.style === "bullet" ? "</ul>" : "</ol>";
      return listHtml;
    },

    convertPreformatted(block) {
      let content = "";

      block.elements.forEach((element) => {
        if (element.type === "text") {
          let textContent = element.text;
          let styledText = textContent;

          // Kiểm tra và áp dụng các style nếu có
          if (element.style) {
            if (element.style.bold) {
              styledText = `<b>${styledText}</b>`; // Bold
            }
            if (element.style.italic) {
              styledText = `<i>${styledText}</i>`; // Italic
            }
            if (element.style.strike) {
              styledText = `<strike>${styledText}</strike>`; // Strike
            }
            if (element.style.code) {
              styledText = `<code>${styledText}</code>`; // Code
            }
          }

          // Nối phần tử text đã được style
          content += styledText;
        }
      });

      return `<pre>${content}</pre>`;
    },

    convertQuote(block) {
      let content = "";

      block.elements.forEach((element) => {
        if (element.type === "text") {
          let textContent = element.text;
          let styledText = textContent;

          // Kiểm tra các kiểu style và áp dụng thẻ HTML tương ứng
          if (element.style) {
            if (element.style.bold) {
              styledText = `<b>${styledText}</b>`; // Bold
            }
            if (element.style.italic) {
              styledText = `<i>${styledText}</i>`; // Italic
            }
            if (element.style.strike) {
              styledText = `<strike>${styledText}</strike>`; // Strike
            }
            if (element.style.code) {
              styledText = `<code>${styledText}</code>`; // Code
            }
          }

          // Nối phần tử text đã được style
          content += styledText;
        }
      });

      return `<blockquote>${content}</blockquote>`;
    },

    convertImage(block) {
      return `<img src="${block.image_url}" alt="${block.alt_text}" title="${
        block.title ? block.title.text : ""
      }" />`;
    },

    convertHeader(block) {
      return `<h${block.level}>${block.text.text}</h${block.level}>`;
    },

    convertStyles(style) {
      // This helper function will convert the style object to inline styles
      let styles = "";
      if (style.bold) styles += "font-weight: bold; ";
      if (style.italic) styles += "font-style: italic; ";
      if (style.strike) styles += "text-decoration: line-through; ";
      if (style.code) styles += "font-family: monospace; ";
      return styles ? ` style="${styles}"` : "";
    },
  },
  computed: {
    editor() {
      return this.$refs.myTextEditor.quill;
    },
  },
};
</script>
<style lang="css" scoped>
.quill-container {
  display: flex;
  width: 100%;
  flex-direction: column;
  margin-top: 150px;
  gap: 20px;
}
.quill-container .editor {
  margin: 0 auto;
  width: 50%;
}
.quill-container .output {
  margin: 0 auto;
  width: 50%;
}
.emoji {
  margin: 0 auto;

  display: flex;
  gap: 20px;
}
.button {
  width: 200px;
  margin: 0 auto;
}
</style>
