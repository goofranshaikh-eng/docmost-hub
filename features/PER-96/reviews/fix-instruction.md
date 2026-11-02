Fix these forge PR review comments (auto-route):

### app comment 4443107169
Minor test gap: this test checks that P_NUM is present, but it does not check the exact characters jsonToText stores. Please add one assertion that the code-block text contains "< P_NUM" (angle bracket, space, then P_NUM) and still contains TEST. Only change this spec file. Do not change codeBlockSearchText or the replace in collaboration.util.ts.
