# 朋友圈九宫格照片新玩法



创意来源于酷安，效果图如下，一张图片分隔成九张小图，每张图片点进去后是一张拼接的长图片

<iframe height=500 width=510 src="./android/image-grid.assets/a28e6c91020475132c0be4b83bf91ff9.mp4"></iframe>

图片分隔成多张图片可以利用Bitmap已有的api实现，给定一张图片，从图片的指定位置生成一张指定宽高的图片

```java
/**
     * Returns a bitmap from the specified subset of the source
     * bitmap. The new bitmap may be the same object as source, or a copy may
     * have been made. It is initialized with the same density and color space
     * as the original bitmap.
     *
     * @param source   The bitmap we are subsetting
     * @param x        The x coordinate of the first pixel in source
     * @param y        The y coordinate of the first pixel in source
     * @param width    The number of pixels in each row
     * @param height   The number of rows
     * @return A copy of a subset of the source bitmap or the source bitmap itself.
     * @throws IllegalArgumentException if the x, y, width, height values are
     *         outside of the dimensions of the source bitmap, or width is <= 0,
     *         or height is <= 0
     */
public static Bitmap createBitmap(@NonNull Bitmap source, int x, int y, int width, int height) {
    return createBitmap(source, x, y, width, height, null, false);
}
```

一张长图片只显示图片的一部分，就需要了解下ImageView提供的几种缩放类型

- center 			当原图的size小于ImageView的size时，保持原图的大小，显示在ImageView的中心。当原图的size大于ImageView的size时，多出来的部分被截掉
- center_inside  当原图的size小于ImageView的size时，不做处理居中显示图片。当原图的size大于ImageView的size时，按照比例缩小原图的宽高，居中显示在ImageView中
- center_crop   当原图的size小于ImageView的size时，则按比例拉升原图的宽和高，填充ImageView居中显示。如果原图size大于ImageView的size，则与center_inside一样，按比例缩小，居中显示在ImageView上。
- matrix  不改变原图的大小，从ImageView的左上角开始绘制，超出部分做剪切处理。
- fit_xy 把图片按照指定的大小在ImageView中显示，拉伸显示图片，不保持原比例，填满ImageView。
- fit_start 把原图按照比例放大缩小到ImageView的高度，显示在ImageView的start（前部/上部）。
- fit_center 把原图按照比例放大缩小到ImageView的高度，显示在ImageView的center（中部/居中显示）。
- fit_end 把原图按照比例放大缩小到ImageView的高度，显示在ImageVIew的end（后部/尾部/底部）



fit方式属于对图片进行拉伸，不适用于当前场景。matrix 需要手动设置参数并不通用。center 方式会截掉多余内容导致图片显示不完整，center_inside  会直接将整个完整图片缩放到ImageView中，剩下的center_crop   方式则很完美，超过后会按照比例缩小并且居中显示。



知道了图片的缩放方式，还有最后一个步骤就是图片合成。单个宫格需要三张图片拼接为一张长图，并且居中的图片需要对应于分隔的小图片。还有一点需要注意，每张图片尽量保证相同大小。



图片拼接代码

```java
/**
     * 三张图片竖直合并为一张图片
     *
     * @param firstBitmap  图片1
     * @param secondBitmap 图片2
     * @param thirdBitmap  图片3
     */
public static Bitmap mergeBitmap(Bitmap firstBitmap, Bitmap secondBitmap, Bitmap thirdBitmap) {
    // 保证3张图片大小一致
    firstBitmap = cropImage(firstBitmap, getImageWidth(), getImageWidth());
    secondBitmap = cropImage(secondBitmap, getImageWidth(), getImageWidth());
    thirdBitmap = cropImage(thirdBitmap, getImageWidth(), getImageWidth());

    Bitmap bitmap = Bitmap.createBitmap(firstBitmap.getWidth(), firstBitmap.getHeight() * 3, firstBitmap.getConfig());

    Canvas canvas = new Canvas(bitmap);
    canvas.drawBitmap(firstBitmap, 0, 0, null);
    canvas.drawBitmap(secondBitmap, 0, firstBitmap.getHeight(), null);
    canvas.drawBitmap(thirdBitmap, 0, firstBitmap.getHeight() + secondBitmap.getHeight(), null);
    return bitmap;
}
```

示例项目效果，会自动保存图片到系统相册，可以直接用来发朋友圈等，注意图片保存的顺序。
<iframe frameborder=0 height=500 width=510 src="./android/image-grid.assets/943358eb246fd42133b59d923d199806.mp4"></iframe>

<iframe frameborder=0 width='200px' height='500px' src="./html/location.html?https://github.com/z-houbin/ImageGrid"></iframe>


朋友圈效果

<iframe height=498 width=510 src="./android/image-grid.assets/0a746a8c328a9b166eb8efc42895553a.mp4"></iframe>
