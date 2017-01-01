---
title: OpenGL Learning(1)  
tags: OpenGL
date: 2016-05-04 00:13:50
---


### 定点和着色器

 >OpenGl只能绘制点、直线及三角形,定义定点时总是以逆时针的顺序排列定点。这称为卷曲顺序。  
 
 >告诉GPU如何绘制数据的东西被称为着色器。着色器分为顶点着色器(vertex shader)和片段着色器(fragment shader)  

![](./opengl2-6.png)

#### 创建第一个顶点着色器

```
attribute vec4 a_Position;

void main() {
	gl_Position = a_Position;
}
```

``OpenGL``会把``gl_Position``中的值当作当前定点的最终位置。

#### 创建第一个片段着色器

```
precision mediump float;

uniform vec4 u_Color;

void main() {
	gl_FragColor = u_Color;
}
```

``OpenGl``会把``gl_FragColor``的值作为当前片段的最终颜色

#### 加载着色器

```
public class TextureResourceReader {
    public static String readTextFileFromResource(Context context, int resId) {
        StringBuilder sb = new StringBuilder();

        try {
            InputStream is = context.getResources().openRawResource(resId);
            InputStreamReader isr = new InputStreamReader(is);
            BufferedReader br = new BufferedReader(isr);
            String nextLine;

            while ((nextLine = br.readLine()) != null) {
                sb.append(nextLine);
                sb.append('\n');
            }
        } catch (IOException e) {
            throw new RuntimeException("Could not open resource: " + resId, e);
        } catch (Resources.NotFoundException e) {
            throw new RuntimeException("Resource not found: " + resId, e);
        }
        return sb.toString();
    }
}
```

### 读入着色器代码

```
@Override protected String readVertexShader() {
        return TextureResourceReader.readTextFileFromResource(mContext, R.raw.simple_vertex_shader);
    }

    @Override protected String readFragmentShader() {
        return TextureResourceReader.readTextFileFromResource(mContext,
                                                              R.raw.simple_fragment_shader);
    }
```

### 编译着色器

```
	public static int compileVertexShader(String shaderCode) {
        return compileShader(GL_VERTEX_SHADER, shaderCode);
    }

    public static int compileFragmentShader(String shaderCode) {
        return compileShader(GL_FRAGMENT_SHADER, shaderCode);
    }
    
	private static int compileShader(int type, String shaderCode) {
        final int shaderObjectId = glCreateShader(type);

        if (shaderObjectId == 0) {
            LogHelper.w(TAG, "Could not create new shader.");
            return 0;
        }

        glShaderSource(shaderObjectId, shaderCode);
        glCompileShader(shaderObjectId);

        final int[] compileStatus = new int[1];
        glGetShaderiv(shaderObjectId, GL_COMPILE_STATUS, compileStatus, 0);

        LogHelper.v(TAG,
                    "Results of compiling source:" + "\n" + shaderCode + "\n" + glGetShaderInfoLog(
                            shaderObjectId));

        if (compileStatus[0] == 0) {
            glDeleteShader(shaderObjectId);

            LogHelper.w(TAG, "Compilation of shader failed");
            return 0;
        }

        return shaderObjectId;
    }
```

### 链接着色器
一个OpenGL程序就是把一个顶点着色器和一个片段着色器链接在一起变成单个对象，顶点着色器和片段着色器总是在一起工作的。片段着色器负责绘制那些组成每个点、直线和三角形的片段；顶点着色器确定在哪里绘制。

```
	public static int linkProgram(int vertextShaderId, int fragmentShaderId) {
        final int programObjectId = glCreateProgram();

        if (programObjectId == 0) {
            LogHelper.w(TAG, "Could not create new program");
            return 0;
        }

        glAttachShader(programObjectId, vertextShaderId);
        glAttachShader(programObjectId, fragmentShaderId);

        glLinkProgram(programObjectId);

        final int[] status = new int[1];
        glGetProgramiv(programObjectId, GL_LINK_STATUS, status, 0);

        LogHelper.v(TAG, "Result of linking program: \n" + glGetProgramInfoLog(programObjectId));

        if (status[0] == 0) {
            glDeleteProgram(programObjectId);
            LogHelper.w(TAG, "Linking of program failed.");
            return 0;
        }

        return programObjectId;
    }
```

### 最后的拼接

* 获取一个``uniform``的位置

```
glGetUniformLocation(program, U_COLOR);
	
```
* 获取属性的位置

```
glGetAttribLocation(program, A_COLOR);
```

* 关联属性和定点数据的数组

```
vertexData.position(0);
//****** very import method *******//
glVertexAttribPointer(aPositionLocation, POSITION_COMPONENT_COUNT, GL_FLOAT, false, STRIDE,
                      vertexData);
```

* 使能顶点数组

```
glEnableVertexAttribArray(aPositionLocation);
```

* 绘制

```
glClear(GL_COLOR_BUFFER_BIT);

//set triangle color
//glUniform4f(uColorLocation, 1.0f, 1.0f, 1.0f, 1.0f);
//draw two triangles
glDrawArrays(GL_TRIANGLE_FAN, 0, 6);

//set line color
//glUniform4f(uColorLocation, 1.0f, 0.0f, 0.0f, 1.0f);
glDrawArrays(GL_LINES, 6, 2);

//draw the first blue mallets
//glUniform4f(uColorLocation, 0.0f, 0.0f, 1.0f, 1.0f);
glDrawArrays(GL_POINTS, 8, 1);

//draw the second red mallets
//glUniform4f(uColorLocation, 1.0f, 0.0f, 0.0f, 1.0f);
glDrawArrays(GL_POINTS, 9, 1);
```

### OpenGl如何把坐标映射到屏幕
OpenGl会把屏幕映射到x,y轴的[-1, 1]范围内，可以给``gl_PointSize``赋值来改变点的大小