Refactoring and fixing massive Objective-C/C function for scanning test images (similar to Scantron).

Initial implementation was a single ~800 line `scan:` function.

``` objectivec
- (id)init;
- (void)scan:(UIImage *)image;
```

Refactored to add 11 helper functions (~45 lines each) with names to describe function of underlying code while also fixing reliability of scanning from ~80% to ~99%:

``` objectivec
- (id)init;

- (void)setHorizontalSpacingFromTotalWidth:(float)totalWidth;

- (void)setVerticalSpacingFromTotalHeight:(float)totalHeight;

-(UIImage *)UIImageFromCVMatRef:(cv::Mat *)cvMatRef;

- (BOOL)imageIsBlack:(cv::Mat)image atCol:(int)col andRow:(int)row;

- (void)findRightColumnEdges:(cv::Mat)binary rightY:(int*)righty leftY:(int*)lefty leftX:(int*)leftx rightX:(int*)rightx;

- (void)findLeftColumnEdges:(cv::Mat)binary rightY:(int*)rightY leftY:(int*)leftY rightX:(int*)rightX leftX:(int*)leftX;

- (void)gradeLeftColumnQuestionsForImage:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2;

- (void)gradeRightColumnQuestion:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2 questionNum:(int)numq;

- (void)findLanguageAndForm:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2;

- (void)parseCfidArea:(cv::Mat)binary answerImage:(cv::Mat)binary2 bottomRightX:(int)bottomRightX rowPositionsY:(NSMutableArray *)rowPositionsY;

- (cv::Mat)setThresholdFromGreyscale:(cv::Mat)greyMat image:(cv::Mat)mat;

- (void)scan:(UIImage *)image;
```
