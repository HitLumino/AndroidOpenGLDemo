
本文主要根据[作者博客](https://blog.csdn.net/junzia/article/details/52793354)以及demo展示，配合文字图片进行相应的解读，纯属笔记。

## render

> 主要绘制三角形，矩形，球等基础形状，可以学习到openglES基本知识。比如varying,uniform attribute等shader限定符基本用法。

### 三角形

#### 着色器语言

> 着色器（Shader）是在GPU上运行的小程序。从名称可以看出，可通过处理它们来处理顶点。此程序使用OpenGL ES SL语言来编写。它是一个描述顶点或像素特性的简单程序。

##### 顶点着色器

>  对于发送给GPU的每一个顶点，都要执行一次顶点着色器。其功能是把每个顶点在虚拟空间中的三维坐标变换为可以在屏幕上显示的二维坐标，并带有用于z-buffer的深度信息。**顶点着色器可以操作的属性有：位置、颜色、纹理坐标，但是不能创建新的顶点。**

![vertex着色器](Note_images/vertex着色器.png)

##### 片元着色器

> 片元着色器计算每个像素的颜色和其它属性。**它通过应用光照值、凹凸贴图，阴影，镜面高光，半透明等处理来计算像素的颜色并输出。**它也可改变像素的深度(z-buffering)或在多个渲染目标被激活的状态下输出多种颜色。一个片元着色器不能产生复杂的效果，因为它只在一个像素上进行操作，而不知道场景的几何形状。

![fragment着色器](Note_images/fragment着色器.png)

代码示例：

```c
private final String vertexShaderCode =
            "attribute vec4 vPosition;" +
                    "uniform mat4 vMatrix;"+
                    "varying  vec4 vColor;"+
                    "attribute vec4 aColor;"+
                    "void main() {" +
                    "  gl_Position = vMatrix*vPosition;" +
                    "  vColor=aColor;"+
                    "}";

    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "varying vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
                    "}";
```

> Notes: 
>
> gl_position`和`gl_FragColor`属于内置变量，分别为定点位置和片元颜色，不需要声明。
>
> 对于`uniform`限定符一般用于对同一组顶点组成的3D物体中各个顶点都相同的量。比如对一组vertex进行矩阵相同的变换。`attribute`限定符可以用来表示不同的值，比如不同的颜色color。varying限定符 主要用来表示从顶点着色器计算出来传到片元着色器中，当作输入值，因而在顶点着色器中，varying 表示需要赋值的量。

#### 投影

> 如果不设置投影矩阵的话，即使在2D图形种，根据坐标值，往往会根据屏幕长宽比出现畸形。
>
> 投影分为2种，正交投影（**orthoM**），透视投影（**frustumM** ）。

![投影](Note_images/投影.png)

* 使用正交投影，物体呈现出来的大小不会随着其距离视点的远近而发生变化。
* 使用透视投影，物体离视点越远，呈现出来的越小。离视点越近，呈现出来的越大。

##### Matrix.setLookAtM解析

```java
public static void setLookAtM(float[] rm, int rmOffset,
            float eyeX, float eyeY, float eyeZ, // 相机所在的位置，摄像机的位置，即观察者眼睛的位置
            float centerX, float centerY, float centerZ, // 摄像机目标点 center of view
            float upX, float upY,float upZ) // 相机的UP向量
```

改变摄像机顶部的方向很显然会改变相机旋转，这样就会会影响到绘制图像的角度。 (0,1,0)：这是相机正对着目标图像。(1,0,0)：左旋90°。(0,0,1)：镜头向上，看屁。

##### 透视矩阵`frustumM`

```java
public static void frustumM(float[] m, int offset,
            float left, float right, float bottom, float top, // 1
            float near, float far) // 2
```

1. [1] 这4个参数会影响图像左右和上下缩放比，所以往往会设置的值分别`-(float) width / height`和`(float) width / height`，top和bottom和top会影响上下缩放比，如果left和right已经设置好缩放，则bottom只需要设置为-1，top设置为1，这样就能保持图像不变形。也可以将left，right 与bottom，top交换比例，即bottom和top设置为` -height/width 和 height/width`, left和right设置为-1和1。 
2. [2] near和far参数稍抽象一点，就是一个立方体的前面和后面，near和far需要结合拍摄相机即观察者眼睛的位置来设置。例如setLookAtM中设置`cx = 0, cy = 0, cz = 10`，near设置的范围需要是小于10才可以看得到绘制的图像，如果大于10，图像就会处于了观察这眼睛的后面，这样绘制的图像就会消失在镜头前，far参数，far参数影响的是立体图形的背面，**far一定比near大，一般会设置得比较大，如果设置的比较小，一旦3D图形尺寸很大，这时候由于far太小，这个投影矩阵没法容纳图形全部的背面，这样3D图形的背面会有部分隐藏掉的。**

最后：由于三角形绘制较为简单，来个完整代码展示基本写法。

```java
public class TriangleColorFull extends Shape {

    private FloatBuffer vertexBuffer,colorBuffer;
    private final String vertexShaderCode =
            "attribute vec4 vPosition;" +
                    "uniform mat4 vMatrix;"+
                    "varying  vec4 vColor;"+
                    "attribute vec4 aColor;"+
                    "void main() {" +
                    "  gl_Position = vMatrix*vPosition;" +
                    "  vColor=aColor;"+
                    "}";

    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "varying vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
                    "}";

    private int mProgram;

    static final int COORDS_PER_VERTEX = 3;
    static float triangleCoords[] = {
            0.5f,  0.5f, 0.0f, // top
            -0.5f, -0.5f, 0.0f, // bottom left
            0.5f, -0.5f, 0.0f  // bottom right
    };

    private int mPositionHandle;
    private int mColorHandle;

    private float[] mViewMatrix=new float[16];
    private float[] mProjectMatrix=new float[16];
    private float[] mMVPMatrix=new float[16];

    //顶点个数
    private final int vertexCount = triangleCoords.length / COORDS_PER_VERTEX;
    //顶点之间的偏移量
    private final int vertexStride = COORDS_PER_VERTEX * 4; // 每个顶点四个字节

    private int mMatrixHandler;

    //设置颜色
    float color[] = {
            0.0f, 1.0f, 0.0f, 1.0f ,
            1.0f, 0.0f, 0.0f, 1.0f,
            0.0f, 0.0f, 1.0f, 1.0f
    };

    public TriangleColorFull(View mView) {
        super(mView);
        //ByteBuffer bb = ByteBuffer.allocateDirect(
              //  triangleCoords.length * 4);
        //bb.order(ByteOrder.nativeOrder());
        //vertexBuffer = bb.asFloatBuffer();
        //vertexBuffer.put(triangleCoords);
        //可以改为：
        vertexBuffer=ByteBuffer.allocateDirect(triangleCoords.length*4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer().put(triangleCoords);
        vertexBuffer.position(0);

        ByteBuffer dd = ByteBuffer.allocateDirect(
                color.length * 4);
        dd.order(ByteOrder.nativeOrder());
        colorBuffer = dd.asFloatBuffer();
        colorBuffer.put(color);
        colorBuffer.position(0);

        int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER,
                vertexShaderCode);
        int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER,
                fragmentShaderCode);

        //创建一个空的OpenGLES程序
        mProgram = GLES20.glCreateProgram();
        //将顶点着色器加入到程序
        GLES20.glAttachShader(mProgram, vertexShader);
        //将片元着色器加入到程序中
        GLES20.glAttachShader(mProgram, fragmentShader);
        //连接到着色器程序
        GLES20.glLinkProgram(mProgram);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {

    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        //计算宽高比
        float ratio=(float)width/height;
        //设置透视投影
        Matrix.frustumM(mProjectMatrix, 0, -ratio, ratio, -1, 1, 3, 10);
        //设置相机位置
        Matrix.setLookAtM(mViewMatrix, 0, 0, 0, 5.0f, 0f, 0f, 0f, 0f, 1.0f, 0.0f);
        //计算变换矩阵
        Matrix.multiplyMM(mMVPMatrix,0,mProjectMatrix,0,mViewMatrix,0);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT| GLES20.GL_DEPTH_BUFFER_BIT);
        //将程序加入到OpenGLES2.0环境
        GLES20.glUseProgram(mProgram);
        //获取变换矩阵vMatrix成员句柄
        mMatrixHandler= GLES20.glGetUniformLocation(mProgram,"vMatrix");
        //指定vMatrix的值
        GLES20.glUniformMatrix4fv(mMatrixHandler,1,false,mMVPMatrix,0);
        //获取顶点着色器的vPosition成员句柄
        mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
        //启用三角形顶点的句柄
        GLES20.glEnableVertexAttribArray(mPositionHandle);
        //准备三角形的坐标数据
        GLES20.glVertexAttribPointer(mPositionHandle, COORDS_PER_VERTEX,
                GLES20.GL_FLOAT, false,
                vertexStride, vertexBuffer);
        //获取片元着色器的vColor成员的句柄
        mColorHandle = GLES20.glGetAttribLocation(mProgram, "aColor");
        //设置绘制三角形的颜色
        GLES20.glEnableVertexAttribArray(mColorHandle);
        GLES20.glVertexAttribPointer(mColorHandle,4,
                GLES20.GL_FLOAT,false,
                0,colorBuffer);
        //绘制三角形
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vertexCount);
        //禁止顶点数组的句柄
        GLES20.glDisableVertexAttribArray(mPositionHandle);
    }

}
```





