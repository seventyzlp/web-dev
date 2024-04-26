---
title: Airsim Custom Sensor
date: 2024-04-26 17:52:50
categories: sim
---

airsim官方文档里面提到”可以很方便的添加自定义传感器“，但是却没有具体提出添加传感器的方法，并且在各种论坛上都没有人讲是怎么改的自定义传感器，于是笔者打算分享下添加自定义传感器的经验。
之后的一切操作都建立在已经成功把airsim部署在UE环境中，编译UE工程不报错的情况下。

添加的是针对四轴飞行器的传感器
传感器是光敏传感器，输出接收到的环境光照强度(lux)

# 第一步：修改SensorBase.hpp
打开SensorBase.hpp后，可以看到一个叫SensorType的枚举类
``` cpp
        enum class SensorType : uint
        {
            Barometer = 1,
            Imu = 2,
            Gps = 3,
            Magnetometer = 4,
            Distance = 5,
            Lidar = 6,
            Luminance = 7, // this is the custom sensor
        };
```
在这个枚举类中，添加新的传感器类型，并且给这个传感器一个编号，这个编号之后在settings.json里面设置传感器类型的时候用来决定是哪个类型的传感器。

# 第二步：修改AirsimSettings.hpp

airsimsettings.hpp 是会根据settings.json里面写的东西对载具和传感器进行初始化，在这里要做的修改是当settings.json里面设置添加了自定义传感器时，要能够识别并且装上去。

其中有这个函数
``` cpp
// creates default sensor list when none specified in json
static void createDefaultSensorSettings(const std::string& simmode_name,
                                        std::map<std::string, std::shared_ptr<SensorSetting>>& sensors)
{
    if (simmode_name == kSimModeTypeMultirotor) {
        sensors["imu"] = createSensorSetting(SensorBase::SensorType::Imu, "imu", true);
        sensors["magnetometer"] = createSensorSetting(SensorBase::SensorType::Magnetometer, "magnetometer", true);
        sensors["gps"] = createSensorSetting(SensorBase::SensorType::Gps, "gps", true);
        sensors["barometer"] = createSensorSetting(SensorBase::SensorType::Barometer, "barometer", true);
        sensors["luminance"] = createSensorSetting(SensorBase::SensorType::Luminance, "luminance", true); // set luminance sensor as default sensor 
    }
    else if (simmode_name == kSimModeTypeCar) {
        sensors["gps"] = createSensorSetting(SensorBase::SensorType::Gps, "gps", true);
    }
    else {
        // no sensors added for other modes
    }
}
```
这里可以实现在进入飞机模式的时候，自动创建新添加的光学传感器。
另外，还需要修改传感器设置的共享指针
```cpp
std::shared_ptr<SensorSetting> sensor_setting;

switch (sensor_type) {
case SensorBase::SensorType::Barometer:
    sensor_setting = std::shared_ptr<SensorSetting>(new BarometerSetting());
    break;
case SensorBase::SensorType::Imu:
    sensor_setting = std::shared_ptr<SensorSetting>(new ImuSetting());
    break;
case SensorBase::SensorType::Gps:
    sensor_setting = std::shared_ptr<SensorSetting>(new GpsSetting());
    break;
case SensorBase::SensorType::Magnetometer:
    sensor_setting = std::shared_ptr<SensorSetting>(new MagnetometerSetting());
    break;
case SensorBase::SensorType::Distance:
    sensor_setting = std::shared_ptr<SensorSetting>(new DistanceSetting());
    break;
case SensorBase::SensorType::Lidar:
    sensor_setting = std::shared_ptr<SensorSetting>(new LidarSetting());
    break;
case SensorBase::SensorType::Luminance: // set new sensor
    sensor_setting = std::shared_ptr<SensorSetting>(new LuminanceSetting());
    break;
default:
    throw std::invalid_argument("Unexpected sensor type");
}
```

 在这里设置的目的是为了在settings.json 设置添加新建的7号传感器后，创建传感器能够认识到这是新添加的传感器，并且初始化传感器。当然，需要自定义"LuminanceSetting()"这个函数来初始化，不过继承自SensorSetting后其实也没什么需要特化的参数，就直接这样写就行
 ```cpp
        struct LuminanceSetting : SensorSetting
        {
        };
```

# 第三步：在Airsim/source/Airlib/include/sensors下新建传感器库

和别的传感器一样，新建一个文件夹叫luminance，两个文件LumianceBase.hpp和LuminanceSimple.hpp

# 第四步：设置传感器获取到的信息类型
在commonStructs.hpp 这个文件中，添加一个结构体作为光敏传感器的数据形式
```cpp
    struct LuminanceSensorData
    {
        float luminance = 0;
        TTimePoint time_stamp = 0;
        LuminanceSensorData()
        {
        }
    };
```

# 第五步：写LuminanceBase.hpp

```cpp
#ifndef msr_airlib_LuminanceBase_hpp
#define msr_airlib_LuminanceBase_hpp

#include "sensors/SensorBase.hpp"

namespace msr
{
	namespace airlib
	{
		class LuminanceBase : public SensorBase
		{
		public:
			LuminanceBase(const std::string& sensor_name = "") : SensorBase(sensor_name)
			{
			}

			virtual void reportState(StateReporter& reporter) override
			{
				UpdatableObject::reportState(reporter);
				reporter.writeValue("Luminance-Lux", output_.luminance);
				
			}

			const LuminanceSensorData& getOutput() const
			{
				return output_;
			}
		protected:
			void setOutput(const LuminanceSensorData& output)
			{
				output_ = output;
			}
		private:
			LuminanceSensorData output_;
			
		};
	}
}

#endif
```
这个就没啥好说的，之后api读取传感器的数据就是用的getOutput这个函数，需要注意一下，其他的话照着别的传感器写就行。

# 第六步：写LumianceSimple.hpp
```cpp
#ifndef msr_airlib_Luminance_hpp
#define msr_airlib_Luminance_hpp

#include "LuminanceBase.hpp"
#include "LuminanceSimpleParams.hpp"
#include "common/DelayLine.hpp"
#include "common/FrequencyLimiter.hpp"

namespace msr
{
	namespace airlib
	{
		class LuminanceSimple : public LuminanceBase
		{
		public:
			LuminanceSimple(const AirSimSettings::LuminanceSetting& setting = AirSimSettings::LuminanceSetting())
				: LuminanceBase(setting.sensor_name)
			{
				params_.initializeFromSettings(setting);

				delay_line_.initialize(params_.update_latency); // init from json data
				freq_limiter_.initialize(params_.update_frequency, params_.startup_delay);
				
			}

			LuminanceSensorData getOutputInternal()
			{
				LuminanceSensorData output;
				params_.luminance = getLuminance();

                                output.luminance = params_luminance;
				output.time_stamp = clock()->nowNanos();
				return output;
			}


			//*** Start: UpdatableState implementation ***//
			virtual void resetImplementation() override
			{
				freq_limiter_.reset();
				delay_line_.reset();

				delay_line_.push_back(getOutputInternal());
			}

			virtual void update() override // just update for the data
			{
				LuminanceBase::update();

				freq_limiter_.update();
				if (freq_limiter_.isWaitComplete()) {
					delay_line_.push_back(getOutputInternal());
				}

				delay_line_.update();
				if (freq_limiter_.isWaitComplete()) {
					setOutput(delay_line_.getOutput());
				}
				
			}

			//*** End: UpdatableState implementation ***//

			virtual ~LuminanceSimple() = default;

			virtual void reportState(StateReporter& reporter) override
			{
				LuminanceBase::reportState(reporter);
				reporter.writeValue("Luminance", params_.luminance);
			}
			
			const LuminanceSimpleParams& getParams() const
			{
				return params_;
			}
		protected:
			virtual float getLuminance() = 0; // use UnrealLuminanceSensor to override
		private:
			LuminanceSimpleParams params_;

			DelayLine<LuminanceSensorData> delay_line_;
			FrequencyLimiter freq_limiter_;
		};
	};
};

#endif
```
注意，这里修改了update()函数的形式，如果说当刷新率比传感器获取信息的数据快的话，会输出上一次传感器所获取到的环境信息。来匹配帧率与认为设置的传感器刷新频率的关系。

其中对于parms_的初始化，和传感器刷新间隔的设置，是在LumianceSimpleParams.hpp里面定义的

# 第七步：写LumianceSimpleParams.hpp

在刚刚创建LumianceBase.hpp和LumianceSimple.hpp的地方创建这个新文件。
```cpp
#ifndef msr_airlib_LumianceSimpleParams_hpp
#define msr_airlib_LumianceSimpleParams_hpp

#include "common/AirSimSettings.hpp"
#include "common/Common.hpp"

namespace msr
{
	namespace airlib
	{
		struct LuminanceSimpleParams
		{
			float luminance = 0; // lux
			real_T update_latency = 0.5f; // sec
			real_T update_frequency = 20; // hz
			real_T startup_delay = 1; // sec  use these to set sample frequece

			void initializeFromSettings(const AirSimSettings::LuminanceSetting& Setting)
			{
				const auto& json = Setting.settings;
				luminance = json.getFloat("Luminance", luminance);
			}
		};
	}
}

#endif
```
其中就能够设置刷新速度和获取时间，并且可以读到settings.json里面对于传感器初始值的设置。如果想要添加的传感器与UE没什么关系，就可以在SensorFactory里面加上，就照着写就行。
```cpp
        // creates one sensor from settings
        virtual std::shared_ptr<SensorBase> createSensorFromSettings(
            const AirSimSettings::SensorSetting* sensor_setting) const
        {
            switch (sensor_setting->sensor_type) {
            case SensorBase::SensorType::Imu:
                return std::shared_ptr<ImuSimple>(new ImuSimple(*static_cast<const AirSimSettings::ImuSetting*>(sensor_setting)));
            case SensorBase::SensorType::Magnetometer:
                return std::shared_ptr<MagnetometerSimple>(new MagnetometerSimple(*static_cast<const AirSimSettings::MagnetometerSetting*>(sensor_setting)));
            case SensorBase::SensorType::Gps:
                return std::shared_ptr<GpsSimple>(new GpsSimple(*static_cast<const AirSimSettings::GpsSetting*>(sensor_setting)));
            case SensorBase::SensorType::Barometer:
                return std::shared_ptr<BarometerSimple>(new BarometerSimple(*static_cast<const AirSimSettings::BarometerSetting*>(sensor_setting)));
            default:
                throw new std::invalid_argument("Unexpected sensor type");
            }
        }
```
但因为笔者想做的传感器需要调用UE里面的物体，getluminance()是纯虚函数，就不能在里面写。

# 第八步：写UnrealSensorFactory.cpp

因为是基于UE的传感器，所以需要在这个SensorFactory里面加新东西。
```cpp
std::shared_ptr<msr::airlib::SensorBase> UnrealSensorFactory::createSensorFromSettings(
    const AirSimSettings::SensorSetting* sensor_setting) const
{
    using SensorBase = msr::airlib::SensorBase;

    switch (sensor_setting->sensor_type) {
    case SensorBase::SensorType::Distance:
        return std::shared_ptr<UnrealDistanceSensor>(new UnrealDistanceSensor(
            *static_cast<const AirSimSettings::DistanceSetting*>(sensor_setting), actor_, ned_transform_));
    case SensorBase::SensorType::Lidar:
        return std::shared_ptr<UnrealLidarSensor>(new UnrealLidarSensor(
            *static_cast<const AirSimSettings::LidarSetting*>(sensor_setting), actor_, ned_transform_));
    case SensorBase::SensorType::Luminance:
        return std::shared_ptr<UnrealLuminanceSensor>(new UnrealLuminanceSensor(
                *static_cast<const AirSimSettings::LuminanceSetting*>(sensor_setting), actor_, ned_transform_));
    default:
        return msr::airlib::SensorFactory::createSensorFromSettings(sensor_setting);
    }
}
```
其中actor就是当前场景中的那个飞机对象，默认设置是BP_FlyingPawn

ned_transform是传感器的位置设置，具体是啥可以调试的时候打断点看

# 第九步：写UnrealLumianceSensor.h和UnrealLumianceSensor.cpp

在UnrealDistanceSensor.cpp边上新建这两个文件
写UnrealLumianceSensor.h
``` cpp
#pragma once

#include "NedTransform.h"
#include "common/AirSimSettings.hpp"
#include "GameFramework/Actor.h"
#include "sensors/luminance/LuminanceSimple.hpp"
#include "Kismet/GameplayStatics.h"
#include "Engine/World.h"
#include "Engine/TextureRenderTarget2D.h"



class UnrealLuminanceSensor : public msr::airlib::LuminanceSimple
{
public:
	typedef msr::airlib::AirSimSettings AirSimSettings;
	
	UnrealLuminanceSensor(const AirSimSettings::LuminanceSetting& setting,
						  AActor* actor, const NedTransform* ned_transform);
	
	virtual float getLuminance() override;
	
private:
	AActor* actor_;
	const NedTransform* ned_transform_;
	float luminancevalue_;

	AActor* LuminanceCamera;
	FFloatProperty* lumproperty;
	UTextureRenderTarget2D* RenderTarget;
};
```

写UnrealLumianceSensor.cpp
```cpp
#include "UnrealLuminanceSensor.h"



float UnrealLuminanceSensor::getLuminance()
{
	// get luminance form actor detector
	luminancevalue_ = lumproperty->GetPropertyValue_InContainer(LuminanceCamera);
	return luminancevalue_;
}

UnrealLuminanceSensor::UnrealLuminanceSensor(const AirSimSettings::LuminanceSetting& setting,
						  AActor* actor, const NedTransform* ned_transform)
							  : LuminanceSimple(setting),actor_(actor),ned_transform_(ned_transform)
{
	TArray<AActor*> LuminanceCameras;
	UWorld* pos = actor_->GetWorld();
	UGameplayStatics::GetAllActorsWithTag(pos, TEXT("LuminanceSensor"), LuminanceCameras); // get actor from tag

	LuminanceCamera = LuminanceCameras[0];
	lumproperty = FindFieldChecked<FFloatProperty>(LuminanceCamera->GetClass(), "Luminance"); // imposible to refresh the actor when update
	luminancevalue_ = lumproperty->GetPropertyValue_InContainer(LuminanceCamera);
	

}
```

这里实现传感器从UE里面读取相关的数据，要注意的是，不能在update()里面设置获取actor或者world指针，不然会崩溃。这里写到的其实就是读取场景中一个有"LuminanceSensor"Tag对象的值，当然，这个对象可以在飞机上，也可以在场景里面的某个地方。

# 第十步：光敏传感器实现
(不是很完善)

传感器本体
使用一个场景捕获组件2d来拍摄一张图，然后在把这张图存储在纹理渲染目标2d里面，actor读取纹理渲染目标，把这张图里面的每个像素的RGB按照人眼亮度曲线算出亮度，然后求平均值得到光敏传感器亮度(lux)。
但是考虑到场景会自动曝光，其实并不是很准，因为看到的图已经经过了后处理。

# 第十一步：改Rpclib的api来实现请求获取传感器数据
在VehicleApiBase.hpp里面加
```cpp
        // Luminance Sensor API
        virtual const LuminanceSensorData& getLuminanceSensorData(const std::string& luminance_sensor_name) const
        {
            auto* luminance_sensor = static_cast<const LuminanceBase*>(findSensorByName(luminance_sensor_name, SensorBase::SensorType::Luminance));
            if (luminance_sensor == nullptr)
                throw VehicleControllerException(Utils::stringf("No Luminance sensor with name %s exist on vehicle", luminance_sensor_name.c_str()));
            return luminance_sensor->getOutput();
        }
```
在RpclibClientBase.hpp里面加
```cpp
        msr::airlib::LuminanceSensorData getLuminanceSensorData(const std::string& luminance_sensor_name = "", const std::string& vehicle_name ="") const;
```
在RpcServerBase.cpp里面加
```cpp
        pimpl_->server.bind("getLuminanceSensorData", [&](const std::string& luminance_sensor_name, const std::string& vehicle_name) -> RpcLibAdaptorsBase::LuminanceSensorData {
            const auto& luminanceSensorData = getVehicleApi(vehicle_name)->getLuminanceSensorData(luminance_sensor_name);
            return RpcLibAdaptorsBase::LuminanceSensorData(luminanceSensorData);
        });
```
在arduCopterApi.hpp里面sendSensors()设置传感器数据的发送形式
```cpp
// send Luminance Sensor Data
const uint count_luminance_sensors = sensors_->size(SensorBase::SensorType::Luminance);
if (count_luminance_sensors != 0) {
    buf << ","
        "\"Luminance\":{"
        "\"LuminanceData\":[";
    const auto& luminance_output = getLuminanceSensorData("");
    buf << luminance_output.luminance <<",";
    buf << "]}";
}
```

如果想用python调用airsim库的话，就在client.py里面加
```python
    def getluminanceSensorData(self, luminance_sensor_name = '', vehicle_name = ''):
        return LuminanceData.from_msgpack(self.client.call('getLuminanceSensorData', luminance_sensor_name, vehicle_name))
```
以及在types.py里面加数据结构
```python
class LuminanceData(MsgpackMixin):
    luminane = 0.0
    time_stamp = np.uint64(0)
```
# 第十一步：设置红色小按钮REC录下数据内容
在MultirotorPawnSimApi.cpp里面修改飞机REC按键的功能
```cpp
std::string MultirotorPawnSimApi::getRecordFileLine(bool is_header_line) const 
{
    std::string common_line = PawnSimApi::getRecordFileLine(is_header_line);
    if (is_header_line) {
        return "Luminance\t";
    }
    const auto& luminance_data = vehicle_api_->getLuminanceSensorData("");

    std::ostringstream ss;
    ss << luminance_data.luminance << "\t";

    return ss.str();

}
```

# 参考
Adding new APIs - AirSim (microsoft.github.io)​microsoft.github.io/AirSim/adding_new_apis/

当使用ros接收传感器数据时，也需要改airsim_ros_pkg哦