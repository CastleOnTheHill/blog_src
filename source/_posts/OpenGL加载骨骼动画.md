---
title: OpenGL加载骨骼动画
date: 2019-07-03 10:54:06
tags:
- opengl
---
一般的模型的加载在[LeanOpenGL](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/ )里已经讲解得比较清楚了，本博文是介绍如何在[LeanOpenGL](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/ )示例`Mesh.h`，和`Model.h`基础之上扩展，使其支持骨骼动画的播放。

## 骨骼动画的原理

![Assimp数据结构](OpenGL加载骨骼动画/assimp%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

带有骨骼动画的模型除了有skin（即一系列的网格），还有骨骼`aiBone`和动画`aiAnimation`。骨骼没有大小和位置，只有一个名字和初始旋转矩阵（决定了骨骼的初始姿态），除此之外，每一个骨骼还存储了它所影响的顶点的ID以及影响的权重；动画`aiAnimation`存储了一系列的关键帧和当前动画的持续时间。关键帧存储的是从初始姿态到当前姿态，所有骨骼要经过的旋转平移和缩放。

在骨骼动画播放的时候，首先根据当前时间找到动画的前一个关键帧和后一个关键帧，然后根据到这两个关键帧的时间距离进行线性插值，得到当前关键帧。再将当前关键帧的旋转平移和缩放应用到所有相关的骨骼上，从而改变当前的骨骼姿态。

![骨骼动画播放](OpenGL加载骨骼动画/%E9%AA%A8%E9%AA%BC%E5%8A%A8%E7%94%BB%E6%92%AD%E6%94%BE.png)

最后在着色器中，将骨骼当前的姿态，按照权重作用到受其影响的顶点上，从而改变顶点的位置。

## 骨骼相关数据的加载

### 扩展`Mesh`中的顶点结构体：

```c++
#define BONE_INFO_NUM 4

struct Vertex {
	// position
	glm::vec3 Position;
	// normal
	glm::vec3 Normal;
    // 这里的注释是因为项目中使用的都是多边形的模型，并没有纹理贴图，所以去掉了
	//// texCoords
	//glm::vec2 TexCoords;
	//// tangent
	//glm::vec3 Tangent;
	//// bitangent
	//glm::vec3 Bitangent;
	glm::ivec4 boneID; //影响骨骼ID

	glm::vec4 boneWeight; // 对应权重

	Vertex() {
		Position = glm::vec3(0.0f, 0.0f, 0.0f);
		Normal = glm::vec3(0.0f, 0.0f, 0.0f);
		boneID = glm::ivec4(-1, 0, 0, 0);
		boneWeight = glm::vec4(0.0f, 0.0f, 0.0f, 0.0f);
	}

	void AddBoneData(uint BoneID, float Weight) {
		for (unsigned int i = 0; i < BONE_INFO_NUM; i++) {
			if (Weight > boneWeight[i]) {
				for (unsigned int j = BONE_INFO_NUM - 1; j > i; j--) {
					boneWeight[j] = boneWeight[j - 1];
					boneID[j] = boneID[j - 1];
				}
				boneWeight[i] = Weight;
				boneID[i] = BoneID;
				break;
			}
		}
	}

	void normalizeBoneWeight() {
		float totalWeight = boneWeight.x + boneWeight.y + boneWeight.z + boneWeight.w;
		boneWeight.x = boneWeight.x / totalWeight;
		boneWeight.y = boneWeight.y / totalWeight;
		boneWeight.z = boneWeight.z / totalWeight;
		boneWeight.w = boneWeight.w / totalWeight;
	}


};
```

顶点记录了影响它的骨骼的ID`boneID`，和对应的权重`boneWeight`，影响顶点的骨骼数量是没有上限的，在这里我们**只取影响最大的4个骨骼**。这里要注意的是，只取最大的四个，最后他们的权重和并不为1，导致在播放骨骼动画的时候，模型会变形，**所以在所有骨骼都处理完毕之后，要对每一个顶点执行`normalizeBoneWeight`函数，使它们的权重值和为1**。

### 加载骨骼数据并归一化

在加载完`Mesh`的其他数据的时候，遍历`Mesh`中的所有骨骼，将骨骼的ID和权重添加到对应的顶点的属性中，这里参照`ogldev`教程（参考3）的做法，所有`Mesh`的骨骼是存在一起的，这样方便最后一次性将所有骨骼的姿态传入着色器。

```c++
		// 加载骨骼权重信息到顶点，并将骨骼加入allBones和boneMap
		for (uint i = 0; i < mesh->mNumBones; i++) {
			unsigned int BoneIndex = 0;
			string BoneName(mesh->mBones[i]->mName.data);
			if (boneMap.find(BoneName) == boneMap.end()) {
				// Allocate an index for a new bone
				BoneIndex = numBones;
				numBones++;
				Bone tmpBone;
				tmpBone.boneOffset = mesh->mBones[i]->mOffsetMatrix;
				tmpBone.name = BoneName;
				// 加入map
				boneMap[BoneName] = BoneIndex;
				// 加入allBones
				allBones.push_back(tmpBone);
			}
			else {
				BoneIndex = boneMap[BoneName];
			}
			for (uint j = 0; j < mesh->mBones[i]->mNumWeights; j++) {
				unsigned int vertexID = mesh->mBones[i]->mWeights[j].mVertexId;
				float weight = mesh->mBones[i]->mWeights[j].mWeight;
				// 给vertices添加影响骨骼信息
				vertices[vertexID].AddBoneData(BoneIndex, weight);
			}
		}
		// 所有的骨骼都加上影响权重之后
		for (auto& vertex : vertices) {
			vertex.normalizeBoneWeight();
		}
```

## 骨骼动画的渲染

### 计算当前帧

*以下直接直接复用了`ogldev`教程的源码*

这里设置的动画是循环播放的，所以将当前时间以模型持续时间取模，计算出动画的时间位置。

骨骼是一个树的结构，骨骼树的信息存储在以`Scene->mRootNode`为根节点的树中，如果当前`node`的`mName`不为空，那么它就代表一根骨骼，`node`的`childNode`就是它的子骨骼。因为父骨骼的姿态要影响到子骨骼的姿态（如大腿骨移动，小腿骨骼也要跟着移动），所以要从根节点开始，一层一层递归地计算。

```c++
	void BoneTransform(float TimeInSeconds, vector<Matrix4f>& Transforms)
	{
		Matrix4f Identity;
		Identity.InitIdentity();

		float TicksPerSecond = (float)(pScene->mAnimations[0]->mTicksPerSecond != 0 ? pScene->mAnimations[0]->mTicksPerSecond : 25.0f);
		float TimeInTicks = TimeInSeconds * TicksPerSecond;
		float AnimationTime = fmod(TimeInTicks, (float)pScene->mAnimations[0]->mDuration);

		ReadNodeHeirarchy(AnimationTime, pScene->mRootNode, Identity);

		Transforms.resize(numBones);

		for (uint i = 0; i < numBones; i++) {
			Transforms[i] = allBones[i].FinalTransformation;
		}
	}
```

根据当前时间找出当前的前一帧和后一帧，再根据时间差线性插值，计算出的平移，旋转和缩放并结合父骨骼的变换作用到当前骨骼上，最后将计算好的当前骨骼的姿态作为父骨骼的变换，递归地处理所有子骨骼：

```c++
	void ReadNodeHeirarchy(float AnimationTime, const aiNode* pNode, const Matrix4f& ParentTransform)
	{
		string NodeName(pNode->mName.data);

		const aiAnimation* pAnimation = pScene->mAnimations[0]; //选择动画

		Matrix4f NodeTransformation(pNode->mTransformation);

		const aiNodeAnim* pNodeAnim = FindNodeAnim(pAnimation, NodeName);

		if (pNodeAnim) {
			// Interpolate scaling and generate scaling transformation matrix
			aiVector3D Scaling;
			CalcInterpolatedScaling(Scaling, AnimationTime, pNodeAnim);
			// 因为我们项目中的动画没有涉及到大小形变，且有的模型动画自带x100的放大，播放动画的时候会变得非常的大，所以这里将骨骼动画的形变量忽略
             // 忽略scale影响
			Matrix4f ScalingM;
			//ScalingM.InitIdentity();
			ScalingM.InitScaleTransform(Scaling.x, Scaling.y, Scaling.z);

			// Interpolate rotation and generate rotation transformation matrix
			aiQuaternion RotationQ;
			CalcInterpolatedRotation(RotationQ, AnimationTime, pNodeAnim);
			Matrix4f RotationM = Matrix4f(RotationQ.GetMatrix());

			// Interpolate translation and generate translation transformation matrix
			aiVector3D Translation;
			CalcInterpolatedPosition(Translation, AnimationTime, pNodeAnim);
			Matrix4f TranslationM;
			TranslationM.InitTranslationTransform(Translation.x, Translation.y, Translation.z);

			// Combine the above transformations
			NodeTransformation = TranslationM * RotationM * ScalingM;
		}

		Matrix4f GlobalTransformation = ParentTransform * NodeTransformation;

		if (boneMap.find(NodeName) != boneMap.end()) {
			uint BoneIndex = boneMap[NodeName];
			allBones[BoneIndex].FinalTransformation = globalInverseTransform * GlobalTransformation * allBones[BoneIndex].boneOffset;
		}
		// 递归处理所有子骨骼
		for (uint i = 0; i < pNode->mNumChildren; i++) {
			ReadNodeHeirarchy(AnimationTime, pNode->mChildren[i], GlobalTransformation);
		}
	}
```

最后将计算好的所有的骨骼姿态传入着色器中：

```c++
BoneTransform(time, Transforms);
for (unsigned int i = 0; i < numBones; i++) {
    sprintf(uniformName, "gBones[%d]", i);
    GLuint location = glGetUniformLocation(shader.ID, uniformName);
    glUniformMatrix4fv(location, 1, GL_TRUE, (const GLfloat*)Transforms[i]);
}
```

在着色器中，先根据顶点受影响的骨骼ID和权重，计算出当前顶点受骨骼姿态的影响矩阵`BoneTransform`，顶点在乘以`Model，View，Projection`矩阵前，先乘以`BoneTransform`矩阵。

```c++
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
layout (location = 2) in ivec4 BoneIDs;
layout (location = 3) in vec4 Weights;

...
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

... 
const int MAX_BONES = 100;
uniform mat4 gBones[MAX_BONES];

void main()
{
	mat4 BoneTransform = mat4(1.0);
    BoneTransform = gBones[BoneIDs[0]] * Weights[0];
    BoneTransform     += gBones[BoneIDs[1]] * Weights[1];
    BoneTransform     += gBones[BoneIDs[2]] * Weights[2];
    BoneTransform     += gBones[BoneIDs[3]] * Weights[3];
    vs_out.FragPos = vec3(model * BoneTransform * vec4(aPos, 1.0));
	...
    gl_Position = projection * view * vec4(vs_out.FragPos, 1.0);
}
```

## Code

扩展后的完整的[Mesh](https://github.com/sysu-cg16/Code/blob/master/CGFinalProject/AnimatedMesh.h )和[Model](https://github.com/sysu-cg16/Code/blob/master/CGFinalProject/AnimatedModel.h )

## 参考

- [LeanOpenGL](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/ )
- [OpenGL Skeletal Animation Tutorial](https://www.youtube.com/watch?v=f3Cr8Yx3GGA)
- [Skeletal Animation With Assimp](http://ogldev.atspace.co.uk/www/tutorial38/tutorial38.html )