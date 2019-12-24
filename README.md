# MediaStore.Images.保存图片时生成两张图片的问题 并且在oppo手机上时间是1970-01-01 08:00:00 

## 正确方式如下

使用方式一
```java

        private void saveImage(String originFilePath) {
            File appDir =
                    new File(Environment.getExternalStorageDirectory().getPath() + "/leikebaijing/images");
            if (!appDir.exists()) appDir.mkdirs();
            String fileName;
            long dateTaken = System.currentTimeMillis();
            if (originFilePath.contains("/")) {
                fileName = originFilePath.substring(originFilePath.lastIndexOf("/") + 1);
            } else {
                fileName = dateTaken + "jpg";
            }
            File file = new File(appDir, fileName);
            if (!file.exists()) {
                try {
                    file.createNewFile();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (file.exists()) {
                MediaStoreLocal.insertImage(getContentResolver(), ImageUtil.getBitmapFormPath(file.getAbsolutePath()), fileName, null);
                Intent intent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
                Uri uri = Uri.fromFile(file);
                intent.setData(uri);
                sendBroadcast(intent);
                ToastUtil.toastShortMessage("图片已保存");
            } else {
                ToastUtil.toastShortMessage("图片已过期或不存在");
            }
        }

```

使用方式二 kotlin语言
```kotlin
  
    /**
     * 保存图片到图库
     */
    private fun saveImageToGallery(context: Context, bmp: Bitmap) {
        // 首先保存图片
        var appDir = File(
            Environment.getExternalStorageDirectory(),
            "myqr"
        )
        if (!appDir.exists()) {
            appDir.mkdir()
        }
        val fileName = System.currentTimeMillis().toString() + ".jpg"
        val file = File(appDir, fileName)
//        try {
//            val fos = FileOutputStream(file)
//            bmp.compress(Bitmap.CompressFormat.JPEG, 100, fos)
//            fos.flush()
//            fos.close()
//        } catch (e: FileNotFoundException) {
//            e.printStackTrace()
//        } catch (e: IOException) {
//            e.printStackTrace()
//        }
        MediaStoreLocal.insertImage(context.getContentResolver(), bmp, fileName, null)
        val intent = Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE)
        val uri: Uri = Uri.fromFile(file)
        intent.setData(uri)
        context.sendBroadcast(intent)

        Toast.makeText(context, "保存成功", Toast.LENGTH_SHORT).show()
    }

```





工具类

```java

package com.tencent.qcloud.tim.uukit.utils;

import android.content.ContentResolver;
import android.content.ContentUris;
import android.content.ContentValues;
import android.graphics.Bitmap;
import android.graphics.Matrix;
import android.net.Uri;
import android.provider.MediaStore;
import android.util.Log;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.OutputStream;

/**
 * 来自系统 {@link   MediaStore.Images.Media}
 * 使用下面方式在oppo手机上会有1970的问题
 * MediaStore.Images.Media.insertImage(getContentResolver(), file.getAbsolutePath(), fileName, null);
 */
public class MediaStoreLocal {
    private final static String TAG = "MediaStoreLocal";
    public static final String AUTHORITY = "media";
    private static final String CONTENT_AUTHORITY_SLASH = "content://" + AUTHORITY + "/";

    /**
     * The content:// style URI for the "primary" external storage
     * volume.
     */
    public static final Uri EXTERNAL_CONTENT_URI =
            getContentUri("external");

    /**
     * Get the content:// style URI for the image media table on the
     * given volume.
     *
     * @param volumeName the name of the volume to get the URI for
     * @return the URI to the image media table on the given volume
     */
    public static Uri getContentUri(String volumeName) {
        return Uri.parse(CONTENT_AUTHORITY_SLASH + volumeName +
                "/images/media");
    }

    /**
     * Insert an image and create a thumbnail for it.
     *
     * @param cr          The content resolver to use
     * @param source      The stream to use for the image
     * @param title       The name of the image
     * @param description The description of the image
     * @return The URL to the newly created image, or <code>null</code> if the image failed to be stored
     * for any reason.
     */
    public static final String insertImage(ContentResolver cr, Bitmap source, String title, String description) {
        ContentValues values = new ContentValues();
        values.put(MediaStore.Images.Media.TITLE, title);
        values.put(MediaStore.Images.Media.DISPLAY_NAME, title);
        values.put(MediaStore.Images.Media.DESCRIPTION, description);
        values.put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg");
        // Add the date meta data to ensure the image is added at the front of the gallery
        values.put(MediaStore.Images.Media.DATE_ADDED, System.currentTimeMillis());
        values.put(MediaStore.Images.Media.DATE_TAKEN, System.currentTimeMillis());

        Uri url = null;
        String stringUrl = null;    /* value to be returned */

        try {
            url = cr.insert(EXTERNAL_CONTENT_URI, values);

            if (source != null) {
                OutputStream imageOut = cr.openOutputStream(url);
                try {
                    source.compress(Bitmap.CompressFormat.JPEG, 50, imageOut);
                } finally {
                    imageOut.close();
                }

                long id = ContentUris.parseId(url);
                // Wait until MINI_KIND thumbnail is generated.
                Bitmap miniThumb = MediaStore.Images.Thumbnails.getThumbnail(cr, id,
                        MediaStore.Images.Thumbnails.MINI_KIND, null);
                // This is for backward compatibility.
                Bitmap microThumb = StoreThumbnail(cr, miniThumb, id, 50F, 50F,
                        MediaStore.Images.Thumbnails.MICRO_KIND);
            } else {
                Log.e(TAG, "Failed to create thumbnail, removing original");
                cr.delete(url, null, null);
                url = null;
            }
        } catch (Exception e) {
            Log.e(TAG, "Failed to insert image", e);
            if (url != null) {
                cr.delete(url, null, null);
                url = null;
            }
        }

        if (url != null) {
            stringUrl = url.toString();
        }

        return stringUrl;
    }


    private static final Bitmap StoreThumbnail(ContentResolver cr, Bitmap source, long id, float width, float height, int kind) {
        // create the matrix to scale it
        Matrix matrix = new Matrix();

        float scaleX = width / source.getWidth();
        float scaleY = height / source.getHeight();

        matrix.setScale(scaleX, scaleY);

        Bitmap thumb = Bitmap.createBitmap(source, 0, 0,
                source.getWidth(),
                source.getHeight(), matrix,
                true);

        ContentValues values = new ContentValues(4);
        values.put(MediaStore.Images.Thumbnails.KIND, kind);
        values.put(MediaStore.Images.Thumbnails.IMAGE_ID, (int) id);
        values.put(MediaStore.Images.Thumbnails.HEIGHT, thumb.getHeight());
        values.put(MediaStore.Images.Thumbnails.WIDTH, thumb.getWidth());

        Uri url = cr.insert(MediaStore.Images.Thumbnails.EXTERNAL_CONTENT_URI, values);

        try {
            OutputStream thumbOut = cr.openOutputStream(url);

            thumb.compress(Bitmap.CompressFormat.JPEG, 100, thumbOut);
            thumbOut.close();
            return thumb;
        } catch (FileNotFoundException ex) {
            return null;
        } catch (IOException ex) {
            return null;
        }
    }


}


```

ImageUtil 

```java

package com.tencent.qcloud.tim.uukit.utils;

import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Canvas;
import android.graphics.Matrix;
import android.graphics.Paint;
import android.graphics.PorterDuff;
import android.graphics.PorterDuffXfermode;
import android.graphics.Rect;
import android.graphics.RectF;
import android.media.ExifInterface;
import android.net.Uri;

import com.tencent.qcloud.tim.uukit.TUIKit;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;


public class ImageUtil {

    /**
     * @param outFile 图片的目录路径
     * @param bitmap
     * @return
     */
    public static File storeBitmap(File outFile, Bitmap bitmap) {
        // 检测是否达到存放文件的上限
        if (!outFile.exists() || outFile.isDirectory()) {
            outFile.getParentFile().mkdirs();
        }
        FileOutputStream fOut = null;
        try {
            outFile.deleteOnExit();
            outFile.createNewFile();
            fOut = new FileOutputStream(outFile);
            bitmap.compress(Bitmap.CompressFormat.JPEG, 100, fOut);
            fOut.flush();
        } catch (IOException e1) {
            outFile.deleteOnExit();
        } finally {
            if (null != fOut) {
                try {
                    fOut.close();
                } catch (IOException e) {
                    e.printStackTrace();
                    outFile.deleteOnExit();
                }
            }
        }
        return outFile;
    }

    public static Bitmap getBitmapFormPath(Uri uri) {
        Bitmap bitmap = null;
        try {
            InputStream input = TUIKit.getAppContext().getContentResolver().openInputStream(uri);
            BitmapFactory.Options onlyBoundsOptions = new BitmapFactory.Options();
            onlyBoundsOptions.inJustDecodeBounds = true;
            onlyBoundsOptions.inDither = true;//optional
            onlyBoundsOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;//optional
            BitmapFactory.decodeStream(input, null, onlyBoundsOptions);
            input.close();
            int originalWidth = onlyBoundsOptions.outWidth;
            int originalHeight = onlyBoundsOptions.outHeight;
            if ((originalWidth == -1) || (originalHeight == -1))
                return null;
            //图片分辨率以480x800为标准
            float hh = 800f;//这里设置高度为800f
            float ww = 480f;//这里设置宽度为480f
            int degree = getBitmapDegree(uri);
            if (degree == 90 || degree == 270) {
                hh = 480;
                ww = 800;
            }
            //缩放比。由于是固定比例缩放，只用高或者宽其中一个数据进行计算即可
            int be = 1;//be=1表示不缩放
            if (originalWidth > originalHeight && originalWidth > ww) {//如果宽度大的话根据宽度固定大小缩放
                be = (int) (originalWidth / ww);
            } else if (originalWidth < originalHeight && originalHeight > hh) {//如果高度高的话根据宽度固定大小缩放
                be = (int) (originalHeight / hh);
            }
            if (be <= 0)
                be = 1;
            //比例压缩
            BitmapFactory.Options bitmapOptions = new BitmapFactory.Options();
            bitmapOptions.inSampleSize = be;//设置缩放比例
            bitmapOptions.inDither = true;//optional
            bitmapOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;//optional
            input = TUIKit.getAppContext().getContentResolver().openInputStream(uri);
            bitmap = BitmapFactory.decodeStream(input, null, bitmapOptions);

            input.close();
            compressImage(bitmap);
            bitmap = rotateBitmapByDegree(bitmap, degree);

        } catch (Exception e) {
            e.printStackTrace();
        }
        return bitmap;//再进行质量压缩
    }

    public static Bitmap getBitmapFormPath(String path) {
        return getBitmapFormPath(Uri.fromFile(new File(path)));
    }

    public static Bitmap compressImage(Bitmap image) {

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        image.compress(Bitmap.CompressFormat.JPEG, 100, baos);//质量压缩方法，这里100表示不压缩，把压缩后的数据存放到baos中
        int options = 100;
        while (baos.toByteArray().length / 1024 > 100) {  //循环判断如果压缩后图片是否大于100kb,大于继续压缩
            baos.reset();//重置baos即清空baos
            //第一个参数 ：图片格式 ，第二个参数： 图片质量，100为最高，0为最差  ，第三个参数：保存压缩后的数据的流
            image.compress(Bitmap.CompressFormat.JPEG, options, baos);//这里压缩options%，把压缩后的数据存放到baos中
            options -= 10;//每次都减少10
        }
        ByteArrayInputStream isBm = new ByteArrayInputStream(baos.toByteArray());//把压缩后的数据baos存放到ByteArrayInputStream中
        Bitmap bitmap = BitmapFactory.decodeStream(isBm, null, null);//把ByteArrayInputStream数据生成图片
        return bitmap;
    }

    /**
     * 读取图片的旋转的角度
     */
    public static int getBitmapDegree(Uri uri) {
        int degree = 0;
        try {
            // 从指定路径下读取图片，并获取其EXIF信息
            ExifInterface exifInterface = new ExifInterface(FileUtil.getPathFromUri(uri));
            // 获取图片的旋转信息
            int orientation = exifInterface.getAttributeInt(ExifInterface.TAG_ORIENTATION,
                    ExifInterface.ORIENTATION_NORMAL);
            switch (orientation) {
                case ExifInterface.ORIENTATION_ROTATE_90:
                    degree = 90;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_180:
                    degree = 180;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_270:
                    degree = 270;
                    break;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return degree;
    }

    /**
     * 读取图片的旋转的角度
     */
    public static int getBitmapDegree(String fileName) {
        int degree = 0;
        try {
            // 从指定路径下读取图片，并获取其EXIF信息
            ExifInterface exifInterface = new ExifInterface(fileName);
            // 获取图片的旋转信息
            int orientation = exifInterface.getAttributeInt(ExifInterface.TAG_ORIENTATION,
                    ExifInterface.ORIENTATION_NORMAL);
            switch (orientation) {
                case ExifInterface.ORIENTATION_ROTATE_90:
                    degree = 90;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_180:
                    degree = 180;
                    break;
                case ExifInterface.ORIENTATION_ROTATE_270:
                    degree = 270;
                    break;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return degree;
    }

    /**
     * 将图片按照某个角度进行旋转
     *
     * @param bm     需要旋转的图片
     * @param degree 旋转角度
     * @return 旋转后的图片
     */
    public static Bitmap rotateBitmapByDegree(Bitmap bm, int degree) {
        Bitmap returnBm = null;

        // 根据旋转角度，生成旋转矩阵
        Matrix matrix = new Matrix();
        matrix.postRotate(degree);
        try {
            // 将原始图片按照旋转矩阵进行旋转，并得到新的图片
            returnBm = Bitmap.createBitmap(bm, 0, 0, bm.getWidth(), bm.getHeight(), matrix, true);
        } catch (OutOfMemoryError e) {
        }
        if (returnBm == null) {
            returnBm = bm;
        }
        if (bm != returnBm) {
            bm.recycle();
        }
        return returnBm;
    }

    public static int[] getImageSize(String path) {
        int size[] = new int[2];
        try {
            BitmapFactory.Options onlyBoundsOptions = new BitmapFactory.Options();
            onlyBoundsOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeFile(path, onlyBoundsOptions);
            int originalWidth = onlyBoundsOptions.outWidth;
            int originalHeight = onlyBoundsOptions.outHeight;
            //size[0] = originalWidth;
            //size[1] = originalHeight;

            int degree = getBitmapDegree(path);
            if (degree == 0) {
                size[0] = originalWidth;
                size[1] = originalHeight;
            } else {
                //图片分辨率以480x800为标准
                float hh = 800f;//这里设置高度为800f
                float ww = 480f;//这里设置宽度为480f
                if (degree == 90 || degree == 270) {
                    hh = 480;
                    ww = 800;
                }
                //缩放比。由于是固定比例缩放，只用高或者宽其中一个数据进行计算即可
                int be = 1;//be=1表示不缩放
                if (originalWidth > originalHeight && originalWidth > ww) {//如果宽度大的话根据宽度固定大小缩放
                    be = (int) (originalWidth / ww);
                } else if (originalWidth < originalHeight && originalHeight > hh) {//如果高度高的话根据宽度固定大小缩放
                    be = (int) (originalHeight / hh);
                }
                if (be <= 0)
                    be = 1;
                BitmapFactory.Options bitmapOptions = new BitmapFactory.Options();
                bitmapOptions.inSampleSize = be;//设置缩放比例
                bitmapOptions.inDither = true;//optional
                bitmapOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;//optional
                Bitmap bitmap = BitmapFactory.decodeFile(path, bitmapOptions);
                bitmap = rotateBitmapByDegree(bitmap, degree);
                size[0] = bitmap.getWidth();
                size[1] = bitmap.getHeight();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return size;

    }

    public static int[] getImageSize(Uri uri) {
        int size[] = new int[2];
        try {
            InputStream is = TUIKit.getAppContext().getContentResolver()
                    .openInputStream(uri);
            BitmapFactory.Options onlyBoundsOptions = new BitmapFactory.Options();
            onlyBoundsOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeStream(is, null, onlyBoundsOptions);
            int originalWidth = onlyBoundsOptions.outWidth;
            int originalHeight = onlyBoundsOptions.outHeight;
            //size[0] = originalWidth;
            //size[1] = originalHeight;


            int degree = getBitmapDegree(uri);
            if (degree == 0) {
                size[0] = originalWidth;
                size[1] = originalHeight;
            } else {

                //图片分辨率以480x800为标准
                float hh = 800f;//这里设置高度为800f
                float ww = 480f;//这里设置宽度为480f
                if (degree == 90 || degree == 270) {
                    hh = 480;
                    ww = 800;
                }
                //缩放比。由于是固定比例缩放，只用高或者宽其中一个数据进行计算即可
                int be = 1;//be=1表示不缩放
                if (originalWidth > originalHeight && originalWidth > ww) {//如果宽度大的话根据宽度固定大小缩放
                    be = (int) (originalWidth / ww);
                } else if (originalWidth < originalHeight && originalHeight > hh) {//如果高度高的话根据宽度固定大小缩放
                    be = (int) (originalHeight / hh);
                }
                if (be <= 0)
                    be = 1;
                BitmapFactory.Options bitmapOptions = new BitmapFactory.Options();
                bitmapOptions.inSampleSize = be;//设置缩放比例
                bitmapOptions.inDither = true;//optional
                bitmapOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;//optional
                is = TUIKit.getAppContext().getContentResolver()
                        .openInputStream(uri);
                Bitmap bitmap = BitmapFactory.decodeStream(is, null, bitmapOptions);
                bitmap = rotateBitmapByDegree(bitmap, degree);
                size[0] = bitmap.getWidth();
                size[1] = bitmap.getHeight();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return size;

    }

    public static CopyImageInfo copyImage(String path, String dir) {
        CopyImageInfo info = new CopyImageInfo();
        if (null == path) {
            return null;
        }
        try {
            int index = path.lastIndexOf(".");
            String fileType = "";
            if (index >= 0) {
                fileType = path.substring(index + 1);
            }
            String newFileName = dir + File.separator + System.currentTimeMillis() + "." + fileType;
            InputStream is = new FileInputStream(new File(path));
            int degree = getBitmapDegree(path);
            File file = new File(newFileName);
            BitmapFactory.Options onlyBoundsOptions = new BitmapFactory.Options();
            onlyBoundsOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeStream(is, null, onlyBoundsOptions);
            info.setWidth(onlyBoundsOptions.outWidth);
            info.setHeight(onlyBoundsOptions.outHeight);

            //没有旋转，直接copy
            if (degree == 0) {
                is = new FileInputStream(new File(path));
                OutputStream os = new FileOutputStream(file);
                byte bt[] = new byte[1024 * 10];
                int c;
                while ((c = is.read(bt)) > 0) {
                    os.write(bt, 0, c);
                }
                is.close();
                os.close();
            } else {
                int ww = 400;
                int hh = 800;
                if (degree == 90 || degree == 270) {
                    ww = 800;
                    hh = 400;
                }
                int be = 1;//be=1表示不缩放
                if (info.width > info.height && info.width > ww) {//如果宽度大的话根据宽度固定大小缩放
                    be = info.width / ww;
                } else if (info.width < info.height && info.height > hh) {//如果高度高的话根据宽度固定大小缩放
                    be = info.height / hh;
                }
                if (be <= 0) {
                    be = 1;
                }
                //比例压缩
                BitmapFactory.Options bitmapOptions = new BitmapFactory.Options();
                bitmapOptions.inSampleSize = be;//设置缩放比例
                bitmapOptions.inDither = true;//optional
                bitmapOptions.inPreferredConfig = Bitmap.Config.ARGB_8888;//optional
                is = new FileInputStream(new File(path));
                Bitmap bitmap = BitmapFactory.decodeStream(is, null, bitmapOptions);
                bitmap = rotateBitmapByDegree(bitmap, degree);
                info.setWidth(bitmap.getWidth());
                info.setHeight(bitmap.getHeight());
                storeBitmap(file, bitmap);
            }
            info.setPath(newFileName);
            return info;
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 转换图片成圆形
     *
     * @param bitmap
     *            传入Bitmap对象
     * @return
     */
    public static Bitmap toRoundBitmap(Bitmap bitmap) {
        int width = bitmap.getWidth();
        int height = bitmap.getHeight();
        float roundPx;
        float left, top, right, bottom, dst_left, dst_top, dst_right, dst_bottom;
        if (width <= height) {
            roundPx = width / 2;
            left = 0;
            top = 0;
            right = width;
            bottom = width;
            height = width;
            dst_left = 0;
            dst_top = 0;
            dst_right = width;
            dst_bottom = width;
        } else {
            roundPx = height / 2;
            float clip = (width - height) / 2;
            left = clip;
            right = width - clip;
            top = 0;
            bottom = height;
            width = height;
            dst_left = 0;
            dst_top = 0;
            dst_right = height;
            dst_bottom = height;
        }

        Bitmap output = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(output);

        final int color = 0xff424242;
        final Paint paint = new Paint();
        final Rect src = new Rect((int) left, (int) top, (int) right, (int) bottom);
        final Rect dst = new Rect((int) dst_left, (int) dst_top, (int) dst_right, (int) dst_bottom);
        final RectF rectF = new RectF(dst);

        paint.setAntiAlias(true);// 设置画笔无锯齿

        canvas.drawARGB(0, 0, 0, 0); // 填充整个Canvas
        paint.setColor(color);

        // 以下有两种方法画圆,drawRounRect和drawCircle
        // canvas.drawRoundRect(rectF, roundPx, roundPx, paint);// 画圆角矩形，第一个参数为图形显示区域，第二个参数和第三个参数分别是水平圆角半径和垂直圆角半径。
        canvas.drawCircle(roundPx, roundPx, roundPx, paint);

        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));// 设置两张图片相交时的模式,参考http://trylovecatch.iteye.com/blog/1189452
        canvas.drawBitmap(bitmap, src, dst, paint); //以Mode.SRC_IN模式合并bitmap和已经draw了的Circle

        return output;
    }

    public static class CopyImageInfo {
        String path;
        int width;
        int height;

        public String getPath() {
            return path;
        }

        public void setPath(String path) {
            this.path = path;
        }

        public int getWidth() {
            return width;
        }

        public void setWidth(int width) {
            this.width = width;
        }

        public int getHeight() {
            return height;
        }

        public void setHeight(int height) {
            this.height = height;
        }
    }
}

```

