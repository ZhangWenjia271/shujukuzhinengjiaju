# 数据库智能家居系统
## 前期准备
### 调用函数
```python
from datetime import datetime
from typing import List, Optional
from fastapi import FastAPI, Depends, HTTPException
from fastapi.responses import JSONResponse
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, ForeignKey, func
from sqlalchemy.orm import sessionmaker, declarative_base, Session
from sqlalchemy.event import listens_for
from pydantic import BaseModel, ConfigDict, Field
import random
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
from sqlalchemy import func
```

### 数据库连接
```python
DATABASE_URL = "sqlite:///./smart_home.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

```python
app = FastAPI(title="智能家居API")
```

### 自定义响应类，确保中文正确显示
```python
class UTF8JSONResponse(JSONResponse):  
    media_type = "application/json; charset=utf-8"
```

---

## 建立数据库
### Pydantic模型定义
```python
class UserBase(BaseModel):
    name: str
    phone: str
    email: str

class UserCreate(UserBase):
    password: str

class User(UserBase):
    id: int
    model_config = ConfigDict(from_attributes=True)

class DeviceBase(BaseModel):
    name: str
    type: str
    location: str
    status: str
    user_id: int

class Device(DeviceBase):
    id: int
    model_config = ConfigDict(from_attributes=True)

class SecurityLogBase(BaseModel):
    device_id: int
    event_type: str
    event_time: datetime
    description: str

class SecurityLog(SecurityLogBase):
    id: int
    model_config = ConfigDict(from_attributes=True)

class EnergyConsumptionBase(BaseModel):
    device_id: int
    consumption: float
    record_time: datetime

class EnergyConsumption(EnergyConsumptionBase):
    id: int
    model_config = ConfigDict(from_attributes=True)

class HouseBase(BaseModel):
    area: float = Field(..., gt=0, description="房屋面积（平方米）")
    people_count: int = Field(..., gt=0, description="居住人数")
    user_id: int = Field(..., description="关联用户ID")

class HouseCreate(HouseBase):
    pass

class House(HouseBase):
    id: int
    house_type: str = Field(..., description="房型（小型/中型/大型）")
    model_config = ConfigDict(from_attributes=True)
```

### SQLAlchemy模型定义
```python
class DBUser(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    phone = Column(String)
    email = Column(String)
    password = Column(String)

class DBDevice(Base):
    __tablename__ = "devices"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String)
    type = Column(String)
    location = Column(String)
    status = Column(String)
    user_id = Column(Integer, ForeignKey("users.id"))

class DBSecurityLog(Base):
    __tablename__ = "security_logs"
    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(Integer, ForeignKey("devices.id"))
    event_type = Column(String)
    event_time = Column(DateTime)
    description = Column(String)

class DBEnergyConsumption(Base):
    __tablename__ = "energy_consumptions"
    id = Column(Integer, primary_key=True, index=True)
    device_id = Column(Integer, ForeignKey("devices.id"))
    consumption = Column(Float)
    record_time = Column(DateTime)

class DBHouse(Base):
    __tablename__ = "houses"
    id = Column(Integer, primary_key=True, index=True)
    area = Column(Float)
    people_count = Column(Integer)
    house_type = Column(String)  # 自动计算的房型
    user_id = Column(Integer, ForeignKey("users.id"))
```

## 自动计算房型的监听函数
```python
@listens_for(DBHouse, 'before_insert')
@listens_for(DBHouse, 'before_update')
def calculate_house_type(mapper, connection, target):
    """根据面积自动计算房型"""
    if target.area < 60:
        target.house_type = "小型"
    elif 60 <= target.area < 120:
        target.house_type = "中型"
    else:
        target.house_type = "大型"
```

---

## 创建数据库表
```python
Base.metadata.create_all(bind=engine)
```

## 依赖：获取数据库会话
```python
def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## 根路径处理
```python
@app.get("/", include_in_schema=False, response_class=UTF8JSONResponse)
def root():
    return {
        "status": "success",
        "message": "智能家居数据管理API服务运行中",
        "docs": "/docs",
        "redoc": "/redoc"
    }
```

---

## 随机生成数据
### 生成10条用户数据
```python
def create_dummy_users(db: Session):
    for _ in range(10):
        user = DBUser(
            name=f"User{random.randint(1, 1000)}",
            phone=f"138{random.randint(10000000, 99999999)}",
            email=f"user{random.randint(1, 1000)}@example.com",
            password="password123"
        )
        db.add(user)
    db.commit()
```

### 生成10条设备数据
```python
def create_dummy_devices(db: Session):
    users = db.query(DBUser).all()  # 获取所有用户
    if not users:
        print("No users found in the database.")
        return  # 如果没有用户，直接返回

    for _ in range(10):  # 生成10个设备
        device = DBDevice(
            name=f"Device{random.randint(1, 1000)}",  # 设备名称
            type=random.choice(["Light", "Fan", "AC", "Heater"]),  # 随机选择设备类型
            location=f"Room{random.randint(1, 5)}",  # 随机分配位置
            status=random.choice(["On", "Off"]),  # 随机设置设备状态
            user_id=random.choice([user.id for user in users])  # 随机分配用户ID
        )
        db.add(device)
    db.commit()  # 提交所有设备数据
```

### 生成60条安防日志数据
```python
from datetime import datetime, timedelta
import random

def create_dummy_security_logs(db: Session):
    devices = db.query(DBDevice).all()  # 获取所有设备
    if not devices:
        print("No devices found in the database.")
        return  # 如果没有设备，直接返回

    # 增加某些设备的出现频率
    for _ in range(60):  # 增加事件的数量，确保每个设备在某个时间段出现多次
        device = random.choice(devices)  # 随机选择设备
        event_time = datetime.now() - timedelta(hours=random.randint(1, 24))  # 随机生成过去24小时内的时间
        log = DBSecurityLog(
            device_id=device.id,
            event_type="Device On",  # 模拟设备开启事件
            event_time=event_time,
            description=f"Event description {random.randint(1, 1000)}"
        )
        db.add(log)
    db.commit()  # 提交所有事件数据
```

### 生成100条能耗数据
```python
def create_dummy_energy_consumptions(db: Session):
    devices = db.query(DBDevice).all()  # 获取所有设备

    # 增加能耗记录数量，确保设备的频率更高
    for _ in range(100):  # 增加记录数量，确保设备有足够的能耗数据
        # 随机选取多个设备（3到4个设备同时使用）
        selected_devices = random.sample(devices, random.randint(3, 4))  # 随机选择3到4个设备

        # 随机生成一个公共的时间段，确保设备的使用时间重叠
        start_time = datetime.now() - timedelta(hours=random.randint(1, 24))
        end_time = start_time + timedelta(hours=random.randint(1, 3))  # 持续时间1到3小时

        # 为每个设备创建能耗记录，确保它们在相同的时间段内使用
        for device in selected_devices:
            consumption = DBEnergyConsumption(
                device_id=device.id,
                consumption=random.uniform(1.0, 10.0),  # 增加能耗范围
                record_time=start_time  # 使用相同的开始时间
            )
            db.add(consumption)

    db.commit()  # 提交所有记录
```

### 生成10条房屋数据
```python
def create_dummy_houses(db: Session):
    users = db.query(DBUser).all()  # 获取所有用户
    for _ in range(10):
        house = DBHouse(
            area=random.randint(50, 200),  # 房屋面积在50到200平方米之间
            people_count=random.randint(1, 5),  # 居住人数1到5人
            user_id=random.choice([user.id for user in users])
        )
        db.add(house)
    db.commit()
```

---

## 在应用启动时调用这些函数来创建虚拟数据
```python
@app.on_event("startup")
def startup():
    db = SessionLocal()
    create_dummy_users(db)
    create_dummy_devices(db)
    create_dummy_security_logs(db)
    create_dummy_energy_consumptions(db)
    create_dummy_houses(db)
    db.close()
```

---

## API接口
### 用户模块：增删改查
```python
@app.post("/users/", response_model=User, response_class=UTF8JSONResponse)
def create_user(user: UserCreate, db: Session = Depends(get_db)):
    """创建用户"""
    db_user = DBUser(**user.model_dump())
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

@app.get("/users/", response_model=List[User], response_class=UTF8JSONResponse)
def get_users(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """查询用户列表（支持分页）"""
    users = db.query(DBUser).offset(skip).limit(limit).all()
    return users

@app.get("/users/{user_id}", response_model=User, response_class=UTF8JSONResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    """根据ID查询用户"""
    user = db.query(DBUser).filter(DBUser.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="用户不存在")
    return user

@app.put("/users/{user_id}", response_model=User, response_class=UTF8JSONResponse)
def update_user(user_id: int, user: UserCreate, db: Session = Depends(get_db)):
    """更新用户信息"""
    db_user = db.query(DBUser).filter(DBUser.id == user_id).first()
    if not db_user:
        raise HTTPException(status_code=404, detail="用户不存在")

    update_data = user.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_user, key, value)

    db.commit()
    db.refresh(db_user)
    return db_user

@app.delete("/users/{user_id}", response_class=UTF8JSONResponse)
def delete_user(user_id: int, db: Session = Depends(get_db)):
    """删除用户"""
    db_user = db.query(DBUser).filter(DBUser.id == user_id).first()
    if not db_user:
        raise HTTPException(status_code=404, detail="用户不存在")

    db.delete(db_user)
    db.commit()
    return {"status": "success", "message": "用户已删除"}
```

### 设备模块：增删改查
```python
@app.post("/devices/", response_model=Device, response_class=UTF8JSONResponse)
def create_device(device: DeviceBase, db: Session = Depends(get_db)):
    """创建设备"""
    db_device = DBDevice(**device.model_dump())
    db.add(db_device)
    db.commit()
    db.refresh(db_device)
    return db_device

@app.get("/devices/", response_model=List[Device], response_class=UTF8JSONResponse)
def get_devices(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """查询设备列表（支持分页）"""
    devices = db.query(DBDevice).offset(skip).limit(limit).all()
    return devices

@app.get("/devices/{device_id}", response_model=Device, response_class=UTF8JSONResponse)
def get_device(device_id: int, db: Session = Depends(get_db)):
    """根据ID查询设备"""
    device = db.query(DBDevice).filter(DBDevice.id == device_id).first()
    if not device:
        raise HTTPException(status_code=404, detail="设备不存在")
    return device

@app.put("/devices/{device_id}", response_model=Device, response_class=UTF8JSONResponse)
def update_device(device_id: int, device: DeviceBase, db: Session = Depends(get_db)):
    """更新设备信息"""
    db_device = db.query(DBDevice).filter(DBDevice.id == device_id).first()
    if not db_device:
        raise HTTPException(status_code=404, detail="设备不存在")

    update_data = device.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_device, key, value)

    db.commit()
    db.refresh(db_device)
    return db_device

@app.delete("/devices/{device_id}", response_class=UTF8JSONResponse)
def delete_device(device_id: int, db: Session = Depends(get_db)):
    """删除设备"""
    db_device = db.query(DBDevice).filter(DBDevice.id == device_id).first()
    if not db_device:
        raise HTTPException(status_code=404, detail="设备不存在")

    db.delete(db_device)
    db.commit()
    return {"status": "success", "message": "设备已删除"}
```

### 安防日志模块：增删改查
```python
@app.post("/security-logs/", response_model=SecurityLog, response_class=UTF8JSONResponse)
def create_security_log(log: SecurityLogBase, db: Session = Depends(get_db)):
    """创建安防日志"""
    db_log = DBSecurityLog(**log.model_dump())
    db.add(db_log)
    db.commit()
    db.refresh(db_log)
    return db_log

@app.get("/security-logs/", response_model=List[SecurityLog], response_class=UTF8JSONResponse)
def get_security_logs(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """查询安防日志列表（支持分页）"""
    logs = db.query(DBSecurityLog).offset(skip).limit(limit).all()
    return logs

@app.get("/security-logs/{log_id}", response_model=SecurityLog, response_class=UTF8JSONResponse)
def get_security_log(log_id: int, db: Session = Depends(get_db)):
    """根据ID查询安防日志"""
    log = db.query(DBSecurityLog).filter(DBSecurityLog.id == log_id).first()
    if not log:
        raise HTTPException(status_code=404, detail="安防日志不存在")
    return log

@app.put("/security-logs/{log_id}", response_model=SecurityLog, response_class=UTF8JSONResponse)
def update_security_log(log_id: int, log: SecurityLogBase, db: Session = Depends(get_db)):
    """更新安防日志"""
    db_log = db.query(DBSecurityLog).filter(DBSecurityLog.id == log_id).first()
    if not db_log:
        raise HTTPException(status_code=404, detail="安防日志不存在")

    update_data = log.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_log, key, value)

    db.commit()
    db.refresh(db_log)
    return db_log

@app.delete("/security-logs/{log_id}", response_class=UTF8JSONResponse)
def delete_security_log(log_id: int, db: Session = Depends(get_db)):
    """删除安防日志"""
    db_log = db.query(DBSecurityLog).filter(DBSecurityLog.id == log_id).first()
    if not db_log:
        raise HTTPException(status_code=404, detail="安防日志不存在")

    db.delete(db_log)
    db.commit()
    return {"status": "success", "message": "安防日志已删除"}
```

### 能耗记录模块：增删改查
```python
@app.post("/energy-consumptions/", response_model=EnergyConsumption, response_class=UTF8JSONResponse)
def create_energy_consumption(consumption: EnergyConsumptionBase, db: Session = Depends(get_db)):
    """创建能耗记录"""
    db_consumption = DBEnergyConsumption(**consumption.model_dump())
    db.add(db_consumption)
    db.commit()
    db.refresh(db_consumption)
    return db_consumption

@app.get("/energy-consumptions/", response_model=List[EnergyConsumption], response_class=UTF8JSONResponse)
def get_energy_consumptions(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """查询能耗记录列表（支持分页）"""
    consumptions = db.query(DBEnergyConsumption).offset(skip).limit(limit).all()
    return consumptions

@app.get("/energy-consumptions/{consumption_id}", response_model=EnergyConsumption, response_class=UTF8JSONResponse)
def get_energy_consumption(consumption_id: int, db: Session = Depends(get_db)):
    """根据ID查询能耗记录"""
    consumption = db.query(DBEnergyConsumption).filter(DBEnergyConsumption.id == consumption_id).first()
    if not consumption:
        raise HTTPException(status_code=404, detail="能耗记录不存在")
    return consumption

@app.put("/energy-consumptions/{consumption_id}", response_model=EnergyConsumption, response_class=UTF8JSONResponse)
def update_energy_consumption(consumption_id: int, consumption: EnergyConsumptionBase, db: Session = Depends(get_db)):
    """更新能耗记录"""
    db_consumption = db.query(DBEnergyConsumption).filter(DBEnergyConsumption.id == consumption_id).first()
    if not db_consumption:
        raise HTTPException(status_code=404, detail="能耗记录不存在")

    update_data = consumption.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_consumption, key, value)

    db.commit()
    db.refresh(db_consumption)
    return db_consumption

@app.delete("/energy-consumptions/{consumption_id}", response_class=UTF8JSONResponse)
def delete_energy_consumption(consumption_id: int, db: Session = Depends(get_db)):
    """删除能耗记录"""
    db_consumption = db.query(DBEnergyConsumption).filter(DBEnergyConsumption.id == consumption_id).first()
    if not db_consumption:
        raise HTTPException(status_code=404, detail="能耗记录不存在")

    db.delete(db_consumption)
    db.commit()
    return {"status": "success", "message": "能耗记录已删除"}
```

### House 模块：增删改查
```python
@app.post("/houses/", response_model=House, response_class=UTF8JSONResponse)
def create_house(house: HouseCreate, db: Session = Depends(get_db)):
    """创建房屋信息"""
    # 验证用户是否存在
    user = db.query(DBUser).filter(DBUser.id == house.user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="关联的用户不存在")

    db_house = DBHouse(**house.model_dump())
    db.add(db_house)
    db.commit()
    db.refresh(db_house)
    return db_house

@app.get("/houses/", response_model=List[House], response_class=UTF8JSONResponse)
def get_houses(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)):
    """查询房屋列表（支持分页）"""
    houses = db.query(DBHouse).offset(skip).limit(limit).all()
    return houses

@app.get("/houses/{house_id}", response_model=House, response_class=UTF8JSONResponse)
def get_house(house_id: int, db: Session = Depends(get_db)):
    """根据ID查询房屋"""
    house = db.query(DBHouse).filter(DBHouse.id == house_id).first()
    if not house:
        raise HTTPException(status_code=404, detail="房屋信息不存在")
    return house

@app.put("/houses/{house_id}", response_model=House, response_class=UTF8JSONResponse)
def update_house(house_id: int, house: HouseCreate, db: Session = Depends(get_db)):
    """更新房屋信息"""
    db_house = db.query(DBHouse).filter(DBHouse.id == house_id).first()
    if not db_house:
        raise HTTPException(status_code=404, detail="房屋信息不存在")

    # 验证用户是否存在（如果修改了user_id）
    if house.user_id is not None and house.user_id != db_house.user_id:
        user = db.query(DBUser).filter(DBUser.id == house.user_id).first()
        if not user:
            raise HTTPException(status_code=404, detail="关联的用户不存在")

    update_data = house.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(db_house, key, value)

    db.commit()
    db.refresh(db_house)
    return db_house

@app.delete("/houses/{house_id}", response_class=UTF8JSONResponse)
def delete_house(house_id: int, db: Session = Depends(get_db)):
    """删除房屋信息"""
    db_house = db.query(DBHouse).filter(DBHouse.id == house_id).first()
    if not db_house:
        raise HTTPException(status_code=404, detail="房屋信息不存在")

    db.delete(db_house)
    db.commit()
    return {"status": "success", "message": "房屋信息已删除"}
```

---

## API数据分析接口

### 设备使用频率和使用时间段分析接口
```python
@app.get("/analytics/devices/usage-frequency", response_class=UTF8JSONResponse)
def get_device_usage_frequency(db: Session = Depends(get_db)):
    """统计设备的使用频率"""
    result = (
        db.query(
            DBDevice.name,
            func.count(DBSecurityLog.id).label("usage_count")
        )
        .join(DBSecurityLog, DBSecurityLog.device_id == DBDevice.id)
        .group_by(DBDevice.name)
        .order_by(func.count(DBSecurityLog.id).desc())
        .all()
    )
    return [{"device_name": name, "usage_count": count} for name, count in result]
```

```python
@app.get("/analytics/devices/simultaneous-usage", response_class=UTF8JSONResponse)
def get_simultaneous_device_usage(db: Session = Depends(get_db)):
    """统计同一时间段内使用频率最高的1个设备"""

    # 获取所有设备开启的事件
    result = (
        db.query(
            DBSecurityLog.event_time,
            DBDevice.name,
            DBSecurityLog.device_id,
        )
        .join(DBDevice, DBSecurityLog.device_id == DBDevice.id)
        .filter(DBSecurityLog.event_type == "Device On")  # 只考虑设备开启事件
        .order_by(DBSecurityLog.event_time)
        .all()
    )

    # 记录设备的开关时间，并按时间段统计
    usage_dict = {}

    for event_time, device_name, device_id in result:
        hour = event_time.hour  # 提取小时作为时间段
        if hour not in usage_dict:
            usage_dict[hour] = {}
        if device_name not in usage_dict[hour]:
            usage_dict[hour][device_name] = 0
        usage_dict[hour][device_name] += 1  # 增加该设备在该时间段的出现次数

    # 筛选出同一时间段内出现次数最多的3个设备
    top_devices_per_hour = {}

    for hour, devices in usage_dict.items():
        sorted_devices = sorted(devices.items(), key=lambda x: x[1], reverse=True)  # 按出现次数排序
        top_devices = sorted_devices[:1]  # 获取出现频率最高的1个设备
        top_devices_per_hour[hour] = top_devices  # 记录每个小时最常用的1个设备

    # 将结果格式化并返回
    simultaneous_usage = []
    for hour, top_devices in top_devices_per_hour.items():
        simultaneous_usage.append({
            "hour": hour,
            "devices": [{"device_name": device[0], "usage_count": device[1]} for device in top_devices]
        })

    return simultaneous_usage
```

```python
@app.get("/analytics/devices/top-energy-users", response_class=UTF8JSONResponse)
def get_top_energy_users(limit: int = 3, db: Session = Depends(get_db)):
    """查询能耗最高的设备（前N名）"""
    subq = (
        db.query(
            DBEnergyConsumption.device_id,
            func.sum(DBEnergyConsumption.consumption).label("total_consumption")
        )
        .group_by(DBEnergyConsumption.device_id)
        .subquery()
    )
    result = (
        db.query(DBDevice.name, subq.c.total_consumption)
        .join(subq, DBDevice.id == subq.c.device_id)
        .order_by(subq.c.total_consumption.desc())
        .limit(limit)
        .all()
    )
    return [{"device_name": name, "total_consumption": total} for name, total in result]
```

### 用户使用习惯分析接口
```python
@app.get("/analytics/users/device-count", response_class=UTF8JSONResponse)
def get_user_device_count(db: Session = Depends(get_db)):
    """统计每个用户的设备数量"""
    result = (
        db.query(DBUser.name, func.count(DBDevice.id).label("device_count"))
        .join(DBDevice, DBUser.id == DBDevice.user_id, isouter=True)
        .group_by(DBUser.id)
        .all()
    )
    return [{"user_name": name, "device_count": count} for name, count in result]
```

```python
@app.get("/analytics/devices/usage-by-hour", response_class=UTF8JSONResponse)
def get_device_usage_by_hour(db: Session = Depends(get_db)):
    """按小时统计每个设备的使用情况"""
    result = (
        db.query(
            DBDevice.name,
            func.extract("hour", DBSecurityLog.event_time).label("hour"),
            func.count(DBSecurityLog.id).label("usage_count")
        )
        .join(DBSecurityLog, DBSecurityLog.device_id == DBDevice.id)
        .group_by(DBDevice.name, "hour")
        .order_by(DBDevice.name, "hour")
        .all()
    )
    return [
        {"device_name": name, "hour": int(hour), "usage_count": count}
        for name, hour, count in result
    ]
```

### 房屋面积影响分析接口
```python
@app.get("/analytics/house-area/device-usage", response_class=UTF8JSONResponse)
def analyze_house_area_device_usage(db: Session = Depends(get_db)):
    """分析房屋面积对设备使用的影响"""

    # Subquery to calculate device count and average energy consumption for each user
    subq = (
        db.query(
            DBDevice.user_id,
            func.count(DBDevice.id).label("device_count"),
            func.avg(DBEnergyConsumption.consumption).label("avg_consumption")
        )
        .join(DBEnergyConsumption, DBDevice.id == DBEnergyConsumption.device_id)
        .group_by(DBDevice.user_id)
        .subquery()
    )

    # Query to get house area, house type, device count, and avg consumption grouped by house type
    result = (
        db.query(
            DBHouse.house_type,
            func.avg(DBHouse.area).label("avg_area"),  # Average area within each house type group
            func.sum(subq.c.device_count).label("total_device_count"),
            func.avg(subq.c.avg_consumption).label("avg_device_consumption")
        )
        .join(subq, DBHouse.user_id == subq.c.user_id)
        .group_by(DBHouse.house_type)
        .order_by(DBHouse.house_type)
        .all()
    )

    return [
        {
            "house_type": item.house_type,
            "avg_area": round(item.avg_area, 2),  # Round area average for each house type
            "total_device_count": item.total_device_count,
            "avg_device_consumption": round(item.avg_device_consumption, 2) if item.avg_device_consumption else 0
        }
        for item in result
    ]
```

### 数据可视化接口：生成不同户型的总能耗统计柱状图
```python
from fastapi import Response, HTTPException
from sqlalchemy.orm import Session
import matplotlib.pyplot as plt
from io import BytesIO
import traceback
from sqlalchemy import func

        # 设置中文字体支持
plt.rcParams["font.family"] = ["SimHei", "WenQuanYi Micro Hei", "Heiti TC"]
plt.rcParams["axes.unicode_minus"] = False  # 解决负号显示问题
```

```python
@app.get("/analytics/houses/total-energy-consumption-chart", response_class=Response)
def get_total_energy_consumption_chart(db: Session = Depends(get_db)):
    """生成不同户型的总能耗统计柱状图"""
    try:
        # 查询每个户型的总能耗数据
        result = (
            db.query(
                DBHouse.house_type,  # 户型类型：小型/中型/大型
                func.sum(DBEnergyConsumption.consumption).label("total_consumption")  # 总能耗
            )
            .join(DBDevice, DBHouse.user_id == DBDevice.user_id)  # 连接DBHouse与DBDevice
            .join(DBEnergyConsumption, DBDevice.id == DBEnergyConsumption.device_id)  # 连接设备与能耗记录
            .group_by(DBHouse.house_type)  # 按户型类型分组
            .order_by(DBHouse.house_type)  # 按户型类型排序
            .all()
        )

        if not result:
            raise HTTPException(status_code=404, detail="没有足够的能耗数据生成图表")

        # 准备图表数据
        house_types = [item[0] for item in result]  # 户型类型
        total_consumptions = [float(item[1]) for item in result]  # 每个户型的总能耗

        # 创建画布
        fig, ax = plt.subplots(figsize=(10, 6))

        # 绘制柱状图
        bars = ax.bar(house_types, total_consumptions, color=['#5DA5DA', '#FAA43A', '#60BD68'], alpha=0.8)

        # 设置图表标题和坐标轴标签
        ax.set_title("不同户型总能耗对比", fontsize=16, pad=20)
        ax.set_xlabel("户型类型", fontsize=14, labelpad=15)
        ax.set_ylabel("总能耗 (kWh)", fontsize=14, labelpad=15)

        # 添加数值标签
        for i, bar in enumerate(bars):
            height = bar.get_height()
            ax.text(
                bar.get_x() + bar.get_width() / 2., height + 0.05,
                f'{height:.2f} kWh', ha='center', va='bottom', fontsize=10
            )

        # 设置y轴范围
        ax.set_ylim(0, max(total_consumptions) * 1.1)

        # 添加网格线
        ax.grid(axis='y', linestyle='--', alpha=0.7)

        # 添加图例说明
        ax.legend([bars[0]], ['总能耗'], loc='upper left')

        # 优化布局
        plt.tight_layout()

        # 将图表保存到内存
        buffer = BytesIO()
        plt.savefig(buffer, format="png", dpi=300)
        buffer.seek(0)
        plt.close()

        # 返回图像内容
        return Response(content=buffer.getvalue(), media_type="image/png")

    except Exception as e:
        # 打印详细错误信息
        print(traceback.format_exc())
        raise HTTPException(status_code=500, detail=f"生成图表失败: {str(e)}")
```

### 其他分析接口
```python
@app.get("/analytics/security/activity-by-hour", response_class=UTF8JSONResponse)
def get_security_activity_by_hour(db: Session = Depends(get_db)):
    """统计每小时的安防事件数量"""
    result = (
        db.query(
            func.extract("hour", DBSecurityLog.event_time).label("hour"),
            func.count(DBSecurityLog.id).label("event_count")
        )
        .group_by("hour")
        .order_by("hour")
        .all()
    )
    return [{"hour": int(hour), "event_count": count} for hour, count in result]
```

```python
@app.get("/analytics/devices/energy-consumption", response_class=UTF8JSONResponse)
def get_device_energy_consumption(db: Session = Depends(get_db)):
    """按设备统计总能耗"""
    result = (
        db.query(
            DBDevice.name,
            func.sum(DBEnergyConsumption.consumption).label("total_consumption")
        )
        .join(DBEnergyConsumption, DBEnergyConsumption.device_id == DBDevice.id)
        .group_by(DBDevice.name)
        .order_by(func.sum(DBEnergyConsumption.consumption).desc())
        .all()
    )
    return [{"device_name": name, "total_consumption": total} for name, total in result]
```

---

## 应用启动
```python
if __name__ == "__main__":
    import uvicorn

    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

---

## API接口的整体截图和几个重要查询的结果截图
![image](https://github.com/user-attachments/assets/ed042bdc-6f8f-4652-923d-e98f3e23a146)
![image](https://github.com/user-attachments/assets/c611b823-698b-4b2b-9fab-7bd4474b410f)
![image](https://github.com/user-attachments/assets/b1c8c888-e193-42f0-b446-f556e99660bf)
![image](https://github.com/user-attachments/assets/65ed8ffe-bcdf-4521-8327-999f0d0611a2)
![image](https://github.com/user-attachments/assets/1e9c11e9-ec93-4e6c-8ec7-2d4439a5014e)
![image](https://github.com/user-attachments/assets/da7c9532-c786-4e7f-b861-1c084bf634af)
![image](https://github.com/user-attachments/assets/c053091d-76ea-4297-9fb1-723218b5234b)
![image](https://github.com/user-attachments/assets/572da758-1972-4d9f-91f2-2c14c2ba4781)

