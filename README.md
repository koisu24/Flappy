Mean Shift
=============
```C++
void exCvMeanShift() {
    Mat img = imread("MyCat.jpg");
    if (img.empty()) exit(-1);
    cout << "----- exCvMeanShift() -----" << endl;

    resize(img, img, Size(256, 256), 0, 0, INTER_AREA);
    imshow("Src", img);
    imwrite("exCvMeanShift_src.jpg", img);

    pyrMeanShiftFiltering(img, img, 8, 16);

    imshow("Dst", img);
    waitKey();
    destroyAllWindows();
    imwrite("exCvMeanShift_dst.jpg", img);
}


class Point5D {
    //Mean Shift 구현을 위한 전용 포인트 클래스

public:
    float x, y, l, u, v;

    //Point5D();
    //~Point5D();

    void accumPt(Point5D);
    void copyPt(Point5D);
    float getColorDist(Point5D);
    float getSpatialDist(Point5D);
    void scalePt(float);
    void setPt(float, float, float, float, float);
    void printPt();
};


void Point5D::accumPt(Point5D Pt)
{
    x += Pt.x;
    y += Pt.y;
    l += Pt.l;
    u += Pt.u;
    v += Pt.v;
}
void Point5D::copyPt(Point5D Pt)
{
    x = Pt.x;
    y = Pt.y;
    l = Pt.l;
    u = Pt.u;
    v = Pt.v;
}
float Point5D::getColorDist(Point5D Pt)
{
    return sqrt(pow(l - Pt.l, 2) + pow(u - Pt.u, 2) + pow(v - Pt.v, 2));
}
float Point5D::getSpatialDist(Point5D Pt)
{
    return sqrt(pow(x - Pt.x, 2) + pow(y - Pt.y, 2));
}

void Point5D::scalePt(float scale)
{
    x *= scale;
    y *= scale;
    l *= scale;
    u *= scale;
    v *= scale;

}

void Point5D::setPt(float px, float py, float pl, float pa, float pb)
{
    x = px;
    y = py;
    l = pl;
    u = pa;
    v = pb;
}

void Point5D::printPt()
{
    cout << x << " " << y << " " << l << " " << u << " " << v << endl;
}


class MeanShift {
public:
    float bw_spatial = 8;
    float bw_color = 16;
    float min_shift_color = 0.1;
    float min_shift_spatial = 0.1;
    int max_steps = 10;
    vector<Mat> img_split;
    MeanShift(float, float, float, float, int);
    void doFiltering(Mat&);
};


MeanShift::MeanShift(float bs, float bc, float msc, float mss, int ms) {
    bw_spatial = bs;
    bw_color = bc;
    max_steps = ms;
    min_shift_color = msc;
    min_shift_spatial = mss;
}

void MeanShift::doFiltering(Mat& img) {
    int height = img.rows;
    int width = img.cols;
    split(img, img_split);

    Point5D pt, pt_prev, pt_cur, pt_sum;
    int pad_left, pad_right, pad_top, pad_bottom;
    size_t n_pt, step;

    for (int row = 0; row < height; row++) {
        for (int col = 0; col < width; col++) {
            pad_left = (col - bw_spatial) > 0 ? (col - bw_spatial) : 0;
            pad_right = (col + bw_spatial) < width ? (col + bw_spatial) : width;
            pad_top = (row - bw_spatial) > 0 ? (row - bw_spatial) : 0;
            pad_bottom = (row + bw_spatial) < height ? (row + bw_spatial) : height;

            pt_cur.setPt(row, col,
                (float)img_split[0].at<uchar>(row, col),
                (float)img_split[1].at<uchar>(row, col),
                (float)img_split[2].at<uchar>(row, col));
            step = 0;
            do {
                pt_prev.copyPt(pt_cur);
                pt_sum.setPt(0, 0, 0, 0, 0);
                n_pt = 0;
                for (int hx = pad_top; hx < pad_bottom; hx++) {
                    for (int hy = pad_left; hy < pad_right; hy++) {
                        pt.setPt(hx, hy,
                            (float)img_split[0].at<uchar>(hx, hy),
                            (float)img_split[1].at<uchar>(hx, hy),
                            (float)img_split[2].at<uchar>(hx, hy));

                         // < Color bandwidth 안에서 측정 >
                        if (pt.getColorDist(pt_cur) < bw_color) {
                            pt_sum.accumPt(pt);
                            n_pt++;
                        }
                    }
                }

                // < 측정결과를 기반으로 현재픽셀 갱신 >
                pt_sum.scalePt(1.0 / n_pt); 
                pt_cur.copyPt(pt_sum);
                step++;
            } while ((pt_cur.getColorDist(pt_prev) > min_shift_color) &&
                (pt_cur.getSpatialDist(pt_prev) > min_shift_spatial) &&
                (step < max_steps));

            // < 결과 픽셀 갱신 >
            img.at<Vec3b>(row, col) = Vec3b(pt_cur.l, pt_cur.u, pt_cur.v);
        }
    }
}

void exMyMeanShift() {
    Mat img = imread("MyCat.jpg");
    if (img.empty()) exit(-1);
    cout << "---------exMyMeanShift()---------------" << endl;

    resize(img, img, Size(256, 256), 0, 0, INTER_AREA);
    imshow("Src", img);
    imwrite("exMyMeanShift_src.jpg", img);

    cvtColor(img, img, COLOR_RGB2Luv);

    MeanShift MSProc(8, 16, 0.1, 0.1, 10);
    MSProc.doFiltering(img);

    cvtColor(img, img, COLOR_Luv2RGB);

    imshow("Dst", img);
    waitKey();
    destroyAllWindows();
    imwrite("exMymeanSHIft_dst.jpg", img);
}

int main() {
    exCvMeanShift();
    exMyMeanShift();
     
   /* Rect rect = Rect(Point(79, 20), Point(540, 563));
    Rect rect2 = Rect(Point(347,148), Point(747, 467));



    Mat result, result2, bg_model, fg_model;
    Mat img = imread("Takahiro4.jpg", 1);
    Mat img2 = imread("Takahiro5.jpg", 1);
    imshow("original", img);
    grabCut(img, result, rect, bg_model, fg_model, 5, GC_INIT_WITH_RECT); 
    compare(result, GC_PR_FGD, result, CMP_EQ);
    //GC_PR_FGD: GrabCut class 전경 픽셀
    //CMP_EQ: Compare 옵션
    Mat mask(img.size(), CV_8UC3, cv::Scalar(255, 255, 255));
    img.copyTo(mask, result); 
    imshow("result", result);
    imshow("mask", mask);

    waitKey(0);
    grabCut(img2, result2, rect2, bg_model, fg_model, 5, GC_INIT_WITH_RECT); //segmentation
    compare(result2, GC_PR_FGD, result2, CMP_EQ);
    Mat mask2(img2.size(), CV_8UC3, cv::Scalar(255, 255, 255));
    img2.copyTo(mask2, result2);
    imshow("original", img2);
    imshow("result2", result2);
    imshow("mask2", mask2);
    waitKey(0);
    destroyAllWindows();
*/
}
```
